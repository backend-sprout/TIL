# Sentinel 
## Sentinal 개요  
       
**`In-Memory Database`에서의 장애 위험성**.       
레디스 프로세스가 다운되면 메모리 내에 저장됐던 데이터는 유실됩니다.      
복제 노드가 있다면 다행히 그 데이터는 복제 노드에 남아있지만,   
**운영 중인 서비스에서 어플리케이션이 마스터에 연결된 상태였다면 다음 과정을 직접 수행해야 합니다.**  
  
1. 복제 노드에 접속해서 REPLICAOF NO ONE 커맨드를 통해 마스터 연결 해제   
2. 어플리케이션 코드에서 레디스 연결 설정을 변경 (마스터 노드의 IP -> 복제 노드의 IP)
3. 배포
    
실제 운영되는 서비스에서 이를 해결하기까지는 오랜 시간이 걸릴 것이고, 이 기간 동안 데이터가 유실될 수도 있습니다.          
어플리케이션이 마스터 노드에 접근할 수 없을 때 데이터를 가져오기 위해 갑자기 많은 커넥션이 RDBMS에 몰려 서비스 장애로까지 이어진 사례도 많습니다.   
이러한 문제를 해결하기 위해서 Redis 는 Sentinal 기능을 지원해줍니다.    
     
## Sentinel 을 이용한 고가용성      
                  
Redis Sentinel은 Redis Cluster 를 사용하지 않을 때 Redis에 고가용성을 제공합니다.             
Redis Sentinel은 또한 `모니터링`, `알림`과 같은 `기타 부수적 작업`을 제공하고 클라이언트를 위한 구성 제공자 역할을 합니다.  
           
**모니터링** 
* Sentinel은 마스터 및 복제본 인스턴스가 예상대로 작동하는지 지속적으로 확인한다.         

**알림**    
* Sentinel은 API를 통해 모니터링되는 Redis 인스턴스 중 하나에 문제가 있음을 알릴 수 있다.          
  
**자동 장애 조치**   
* 마스터가 예상대로 작동하지 않으면        
* Sentinel은 복제본이 마스터로 승격되고 다른 추가 복제본이 새 마스터를 사용하도록 재구성되고       
* Redis 서버를 사용하는 애플리케이션에 사용할 새 주소에 대해 알리는 장애 조치 프로세스를 시작할 수 있다.   
  
**구성 제공자**  
* Sentinel은 클라이언트 서비스 검색을 위한 권한 소스 역할을 한다.   
* 클라이언트는 주어진 서비스를 담당하는 현재 Redis 마스터의 주소를 요청하기 위해 Sentinels에 연결한다.  
* 장애 조치가 발생하면 Sentinels는 새 주소를 보고한다.  
  

## 분산 시스템으로서의 센티넬

Redis Sentinel은 분산 시스템을 지향합니다.  
Redis Sentinel은 여러 Sentinal 프로세스가 있는 노드들을 하나의 논리적 구성으로 실행되도록 설계되어있습니다.  
  
1. 여러 Sentinal이 마스터를 더 이상 사용할 수 없다는 사실에 동의하면 실패 감지가 수행됩니다.      
2. Sentinel 자체는 모든 Sentinel 프로세스가 작동하지 않는 경우에도 계속 작동하므로 시스템이 장애에 대해 견고합니다.     


## Sentinel 구성
Sentinal의 정상적인 기능을 위해서는 적어도 3개의 Sentinal 인스턴스가 필요합니다.   
각 Sentinal 인스턴스는 Redis의 모든 노드를 감시하며 서로 연결되어 있습니다.      
세 대의 Sentinel 노드 중 과반수 이상(quorum)이 동의해야만 페일오버를 시작할 수 있습니다.(홀수 이유)  

![4(4)](https://user-images.githubusercontent.com/50267433/196723347-887b0e66-8821-4e0e-960f-f2a4d0d06393.png)

  
어플리케이션은 마스터나 복제 노드에 직접 연결하지 않고, Sentinel 노드와 연결합니다.      
Sentinel 노드는 어플리케이션에게 현재 마스터의 IP, PORT를 알려주며, 페일오버 이후에는 새로운 마스터의 IP, PORT 정보를 알려줍니다.   

### Failover 과정
 
![5(4)](https://user-images.githubusercontent.com/50267433/196729690-03444555-4530-40f3-8421-d1a18136a3d5.png)

  
마스터에 두 개의 복제 노드가 연결되어 있다고 가정해보겠습니다.     
이때 마스터가 다운되면 이를 감시하고 있던 Sentinel은 마스터에 진짜 연결할 수 없는지에 대한 투표를 시작합니다.   

![6(4)](https://user-images.githubusercontent.com/50267433/196729775-9b7ec16d-c6dd-4111-a766-f97879b3db55.png)
   
이 투표에서 과반수 이상이 동의하면 페일오버를 시작합니다.         
과반수 이상이기에 위 예시에서는 세 개 중 두 개 노드의 찬성을 얻어 페일오버가 가능합니다.         
  
연결이 안되는 마스터에 대한 복제 연결을 끊고, 복제 노드 중 한 개를 선택하여 마스터로 승격시킵니다.        
또 다른 복제 노드는 승격된 마스터 노드에 연결시킵니다.      
만약 다운되었던 마스터 노드가 다시 살아난다면 새로운 마스터에 복제본으로 연결됩니다.  

## Sentinal 실행하기 

```
redis-sentinel /path/to/sentinel.conf
```
* redis-sentinel 명령어를 통해 센티널 인스턴스를 생성할 수 있습니다.  

```
redis-server /path/to/sentinel.conf --sentinel
```
* redis-server 명령어와 `--sentinel`를 통해서도 센티널 인스턴스를 생성할 수 있습니다.   
       
Sentinel을 실행할 때 구성 파일(sentinel.conf)을 사용하는 것은 필수 입니다.       
이 파일은 재시작시 다시 로드되는 현재 상태를 저장하기 위해 시스템에서 사용하기 때문입니다.       
Sentinel은 구성 파일(sentinel.conf)이 제공되지 않거나 구성 파일 경로가 쓰기 불가능하면 시작을 거부합니다.    

Sentinel 사용하면 TCP 포트 26379을 통해 다른 인스턴스와 연결하고 데이터를 수신할 수 있다.   

### 배포하기 전에 Sentinel에 대해 알아야 할 기본 사항
 
* 강력한 배포를 위해서는 최소 3개의 Sentinel 인스턴스가 필요합니다.   
* 3개의 Sentinel 인스턴스는 독립적인 방식으로 실패하는 것으로 여겨지는 컴퓨터 또는 가상 머신에 배치되어야 합니다.  
* Sentinel + Redis 분산 시스템은 Redis가 비동기식 복제를 사용하기 때문에 오류 발생 시 승인된 쓰기가 유지되는 것을 보장하지 않습니다.      
  그러나 쓰기 손실 창을 특정 순간으로 제한하는 Sentinel을 배포하는 방법이 있지만 배포하는 다른 덜 안전한 방법이 있습니다.   
* 클라이언트에서 Sentinel 지원이 필요합니다.   
  인기 있는 클라이언트 라이브러리는 Sentinel을 지원하지만 전부는 아닙니다.
* 개발 환경에서 수시로 테스트하지 않으면 안전한 HA 설정은 없으며 프로덕션 환경에서 테스트할 수 있다면 더 좋습니다.   
  너무 늦었을 때만(당신의 주인이 작동을 멈추는 새벽 3시에) 명백해지는 잘못된 구성이 있을 수 있습니다.
* Sentinel, Docker 또는 기타 형태의 네트워크 주소 변환 또는 포트 매핑은 주의하여 혼합해야 합니다.   
  Docker는 포트 재매핑을 수행하여 다른 Sentinel 프로세스와 마스터의 복제본 목록에 대한 Sentinel 자동 검색을 중단합니다.   

### sentinel.conf

```conf
// sentinel monitor <master-group-name> <ip> <port> <quorum>
// sentinel <option_name> <master_name> <option_value>

sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3 6380 4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```
   
모니터링할 '마스터'를 지정하기만 하면 분리된 각 마스터(복제본이 여러 개 있을 수 있음)에 다른 이름을 지정할 수 있습니다.        
Sentinel은 자동 검색을 통해 복제본에 대한 정보와 그 구성을 자동으로 업데이트합니다  
또한 장애 조치 중에 복제본이 마스터로 승격될 때마다 그리고 새 Sentinel이 발견될 때마다 구성이 다시 작성됩니다.      

```
sentinel monitor <master-group-name> <ip> <port> <quorum>
```
**quorum**  
* 장애 조치 절차를 시작하기 위해 마스터에 연결할 수 없다는 사실에 동의해야 하는 센티넬의 수입니다
* 쿼럼은 오류를 감지하는 데만 사용되기에, 
  실제로 장애 조치를 수행하려면 Sentinel 중 하나가 **장애 조치의 리더로 선출**되고 계속 진행할 수 있는 권한이 있어야 
  이것은 센티넬 프로세스의 대다수가 투표했을 때만 발생 합니다.   
  
예를 들어 5개의 Sentinel 프로세스가 있고 지정된 마스터의 쿼럼이 2로 설정된 경우 다음과 같은 일이 발생합니다.   
* 두 Sentinel이 마스터에 연결할 수 없다는 데 동시에 동의하면 둘 중 하나가 장애 조치를 시작하려고 시도합니다.   
* 연결할 수 있는 Sentinel이 총 3개 이상인 경우 장애 조치가 승인되고 실제로 시작됩니다.

**기타 센티넬 옵션**  
```
sentinel <option_name> <master_name> <option_value>
```
**down-after-millisecondsSentinel**.  
* 다운되었다고 생각되는 인스턴스에 도달할 수 없어야 하는 시간(밀리초)입니다(PING에 응답하지 않거나 오류로 응답함)

**parallel-syncs**   
* 장애 조치 후 동시에 새 마스터를 사용하도록 재구성할 수 있는 복제본 수를 설정합니다.   
* 숫자가 낮을수록 장애 조치 프로세스를 완료하는 데 더 많은 시간이 소요되지만     
  복제본이 이전 데이터를 제공하도록 구성된 경우       
  모든 복제본이 마스터와 동시에 다시 동기화되는 것을 원하지 않을 수 있습니다.    
* 복제 프로세스는 대부분 복제본에 대해 차단되지 않지만 마스터에서 대량 데이터 로드를 중지하는 순간이 있습니다.   
  이 옵션을 값 1로 설정하여 한 번에 하나의 복제본에만 연결할 수 없도록 할 수 있습니다.     
  
  



 





  



