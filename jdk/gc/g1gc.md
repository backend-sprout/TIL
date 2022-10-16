# G1GC(Garbage First Collector)
G1GC는 스레드 정지가 예측 가능한 시간 안에 이루어지는 점진적으로 처리되는 병렬 Compaction GC다.     
Heap의 범위와 현실적인 목표 스레드 정지 시간을 설정하고 GC가 작업을 할 수 있도록 하는 것이 특징이다.     
 
G1GC는 Heap Area 를 `Young` and `Old`로 물리적으로 구분짓던 Generation을 없애고.   
Heap Area 를  `Region` 단위로 나누고 이를 논리적으로 구분하여 사용하고 있다.           
`Region`의 크기는 `1MB ~ 32MB` 로 전체 Heap 사이즈 용량이 2048로 나누어질 수 있도록 결정된다.    
  
메모리 공간이 필요하다면, `Free Region` 을 `Young Generation`이나 `Old Generation`으로 할당한다.   
할당되어 상용되던 Region 이 비어지면 다시 `Free Region` 리스트로 돌아간다.      
   
* Young Generation : Object가 Allocation 되는 Region의 집합
* Old Generation : Object가 Promotion 되는 Region의 집합 

G1GC는 최대한 **살아있는 객체가 적게 들어있는 `Region`을 수집하는 것이 목표다..    
G1GC는 잘개 나누어진 각 `Region` 들을 `Collection Set` 이라는 집합으로 분류하여 GC를 수행한다.    
즉, G1GC 는 'GC 사이클에 모든 Young / Old 영역에 대해 GC를 수행할 필요가 없어진다는 특징'을 가진다.     
(또한 최대한 살아있는 객체가 적은 Region을 수집함으로써 Free Region 을 더 많이 보유하도록 한다) 
  
```
Garbage First 라는 의미는   
Garbage 로만 꽉찬 Region 부터 Collection을 시작한다는 의미로 발견되자 마자 즉각 Collection을 한다. 
```

G1GC 는 파편화된 Region 및 GC 관리를 위해서 `Remeber Set`과 `Collection Set` 을 사용한다.     

- Remember Set (RSet): 객체가 어떤 region에 저장되어 있는지 기록한 자료구조.   
	- Rset 은 Total Heap 메모리의 5% 미만의 크기를 가진다.(대게)     
	- Marking 작업시 trace 일량을 줄여줘 효율을 높이게 된다.(STW 단축)    
- Collection Set (CSet): GC가 수행될 region이 저장되어 있는 자료구조.   
  
## G1GC Cycle
G1GC는 2페이즈를 번갈아 가면서 GC 작업을 수행한다.   
  
* young-only : old 객체를 새로운 공간으로 옮기는 작업.  
* space-reclamation : 공간을 회수하는 작업.   

아래와 같은 프로세스로 이루어진다.    
1. Old Generation 의 점유율이 threshold 값을 넘어서면 young-only 페이즈로 전환된다.    
2. SATB 알고리즘 방식을 통해 객체들의 마킹 작업을 진행한다.    
3. 마킹을 끝내고, 쓰레기 영역을 해지한다.    
4. Space-Reclamation 페이즈로 들어가야 할지 말지를 판단한다.    
5. young/old 가리지 않고 라이브 객체를 적절한 곳으로 대피시킨다(Evacuation).  
   작업 효율이 떨어지게 되면 이 페이즈는 끝나고, 다시 Young-only 페이즈로 전환된다.   
6. 만약 애플리케이션 메모리가 부족한 경우 G1GC는 다른 GC들처럼 Full GC를 수행한다.   

보자 자세한 설명은 아래에서 설명한다.   

## G1 Collector 의 Garbage Collection

G1 Collector 의 Garbage Collection 은 4단계(세부적으로 6단계)로 이루어져있다.     
기본적으로 Allocation 때, Memory 압박이 생기거나 임계값이 초과될 때 마다 GC가 발생한다.    

1. Initial Mark
2. Root Region Scan
3. Concurrent Mark
4. Remark
5. Clean up
6. Copy

### Initial Mark
GC Root 로부터 바로 참조되는 살아있는 객체(roots)를 mark 한다.        
G1GC는 Evacuation 단계(Young GC)로부터 트리거되기 때문에 (Piggy-backend) 오버헤드는 덜하다.    

```
[GC pause (G1 Humongous Allocation) (young) (initial-mark), 0.0021268 secs]
```

### Root Region Scan
Root Region(Survivor region)에 속해 있는 살아있는 모든 객체를 Mark 한다.    

즉, Initial-Mark 의 Survivor 영역을 스캔하여    
Old 영역에 대한 Reference를 구하고 Reference 되고 있는 객체로 Mark를 진행하는 것이다.    

이 단계에서는 STW가 발생하지 않고 애플리케이션 스레드와 동시에 동작하며,   
Survivor region 을 건드리는 다음 Evacuation 단계가 일어나기 전까지 반드시 완료해야한다.    

```
[GC concurrent-root-region-scan-start]
[GC concurrent-root-region-scan-end, 0.--28513 secs]
```

### Concurrent Mark
객체 그래프를 따라   
Java Heap 영역의 살아있는 모든 객체를 식별하여 특별한 비트맵을 이용하여 Mark 를 진행한다.    

또한 모든 Region에 대해서   
가지고 있는 살아있는 객체의 수를 나타내는 live start (Region Liveness)를 계산한다.   

G1GC는 Java Heap 에서 접근 가능한 살아있는 객체를 찾는다.    
그러나, Young GC로 인해서 중간에 작업에 방해를 받을 수 있다.    
이 단계에서는 STW가 발생하지 않고 애플리케이션 스레드와 동시에 동작한다.    

### Remark
이 단계는, STW를 일으키는 단계로 Mark 하는 단계를 완료하는 단계이다.     
애플리케이션 스레드를 잠시 멈추고 Log Buffer에 있던 정보를 참조하여 객체 상태 업데이트를 완료한다.    

SATB(Snapshot-At-The-Beginning) 버퍼를 비우고 아직 추적하지 않은 생존 객체를 찾는다.    
또한 이 단계에서 Phantom Reference, Soft Reference, Weak Reference 와 같은   
일반 Object 와는 다른 Reference Object 를 처리한다.(Referenec Processing).   

### CleanUp / Copy

G1GC는 Free Region과 부분적으로 점유하고 있는 후보군 Region 을 구분하기 위해    
가끔 STW 걸어두고 Region의 생존정도를 측정하고 Rests 를 비운다.   
하지만, Region을 비우고 Free Region 들의 List로 되돌려 놓을때는   
STW를 걸지 않고 애플리케이션과 동시에 처리된다.    

각 로직에 따라 애플리케이션 스레드와 동시에 수행되는 것도 있고, STW 발생도 있다.      
이 단계에서는 GC를 위해 Heap 영역에 있는 모든 살아있는 객체 정보를 조사해둔다.       
  

## IHOP
> IHOP: Initiating Heap Occupancy Percent. 

IHOP는 마킹이 발동되는 기준값(threshold)으로, Old Generation 에 대한 백분율이다.    

```
기본값으로 Heap Area 전체 영역의 45%이지만    
InitiatingHeapOccupancyPercent 라는 JVM 옵션을 통해 변경할 수 있다.    
```

**Adaptive IHOP**
- G1 통계를 계산하며 최적의 IHOP 값을 찾아내 알아서 설정한다.    
-  Adaptive IHOP 기능이 켜져 있을 때  `-XX:InitiatingHeapOccupancyPercent` 옵션을 주면    
  통계 자료가 충분하지 않은 초기 상태에서 이 옵션 값을 초기값으로 활용한다.    
* `-XX:-G1UseAdaptiveIHOP` 옵션으로 Adaptive IHOP 기능을 끌 수 있다.   
	* -   Adaptive 기능을 끄면 통계를 게산하지 않으므로    
	  `-XX:InitiatingHeapOccupancyPercent`로 지정한 IHOP 값을 계속 쓰게 된다.   
-  Adaptive IHOP는 `-XX:G1HeapReservePercent`로 설정된 값 만큼의 버퍼를 제외하고.    
  시작 heap 점유율을 설정한다.   

## SATB(Snapshot-At-The-Beginning) 알고리즘
- G1GC는 SATB(Snapshot-At-The-Beginning) 알고리즘을 써서 마킹 작업을 한다.   
- STAB 를 이용하면 **일시 정지가 일어난 시점 직후의 라이브 객체에만 마킹을 한다.**     
	- 따라서 마킹하는 도중에 죽은 객체도 라이브 객체로 간주하는 보수적인 특징을 가진다.    
	- 비효율적일 것 같지만 Remark 단계의 응답 시간(latency)이 다른 GC에 비해 더 빠른 경향이 있다.   

## Evacuation Failure
G1GC는 GC 대상이 된 Region 중 라이브 객체는 다른 Region 으로 옮기는 작업을 수행한다.   
그런데 만약, **대피시킬 만한 공간이 충분치 않다면 Evacuation Failure(대피 실패)가 발생한다.**       

GC 대상이 된 Region 에서 이미 이동시킨 객체는 새 위치 그대로 유지한다.    
GC 대상이 된 Region 에서 아직 이동시키지 못한 객체는 복사를 진행하지 않는다.(공간 없음).   

단, **이동하지 않은 객체는 참조를 끊지 않도록 조정하여 일단은 무사히 GC 작업을 마치는 방향으로 진행한다.**.     
그러나 이 같은 작업을 진행했음에도 애플리케이션 실행시 Heap 메모리가 부족하다면 Full GC가 예약된다.        
  
## Humongous Object
* 이름 그대로 `커다란 객체` 를 의미하며 한 Region의 절반 이상의 크기를 가진 객체를 말한다.   
* 한 Region 의 절반이 기준이므로, `-XX:G1HeapRegionSize`의 영향을 받는다.  
  
커다란 객체는 아무래도 크기가 있다보니 특별하게 관리가 된다.    

```ascii-art
 영역1  영역2  영역3  영역4
+------+------+------+------+
|######|######|######|###   |
|######|######|######|###   |
+------+------+------+------+
                          ^ 잉여 공간
```

* 연속된 영역을 순차적으로 차지하도록 할당된다.   
* 마지막 꼬리 영역에 남는 공간이 생길 수 있는데, 그 잉여 공간은 사용하지 않는다.(아깝지만).  
	* 즉, 커다란 객체가 회수 될때 까지는 잉여 공간을 사용할 수 없다.   
* 커다란 객체가 할당되면,   
  G1은 IHOP를 확인하고 IHOP가 초과된 상태라면 즉시 강제로(force) Minor GC를 시작한다.  
* 커다란 객체는 Full GC 중에도 옮겨지지 않는데, 이로 인해 조각화가 발생할 수 있다.  
	* 공간이 충분한데도 메모리 부족 상태가 발생할 수 있다.   
	* Full GC가 느려질 수 있다.  
