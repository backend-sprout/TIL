# schedule & serializability
## schedule

**시나리오**
```
A가 B에게 20만원 송금
B가 자신에게 30만원 입금
```
* 1번 케이스 : 송금 완료후 입금 
* 2번 케이스 : 입금 완료후 송금   
* 3번 케이스 : A계좌 뺄샘 후, B 입금 완료, A 송금값 B반영(read는 이때)
* **4번 케이스 : 동시에 읽었기에, 송금 누락 또는 입금 누락(lost update)**   

<img width="837" alt="image" src="https://user-images.githubusercontent.com/50267433/192544480-59820fa4-bc2a-423c-96bb-bf1d3c6a163c.png">
 
이를 각 Operation 들로 나타내면 아래와 같다.    
트랜잭션을 처리하기 위한 각각의 명령 단위를 Operation 이라고 부른다.    

* Schedule 1 : r1(K) -> w1(K) -> r1(H) -> w1(H) -> c1 -> r2(H) -> w2(H) -> c2
* Schedule 2 : r2(H) -> w2(H) -> c2 -> r1(K) -> w1(K) -> r1(H) -> w1(H) -> c1
* Schedule 3 : r1(k) -> w1(k) -> r2(H) -> w2(H) -> c2 -> r1(H) -> w1(H) -> c1
* Schedule 4 : r1(k) -> w1(k) -> r1(H) -> r2(H) -> w2(H) -> c2 -> w1(H) -> c1

Scehdule 은 여러 transaction 들이 동시에 실행될 때,      
각 transaction 애 속한 operation 들의 실행 순서를 의미한다.         
특징으로, 각 transaction 내의 operations 들의 순서는 바뀌지 않는다.   

### Serial Schedule

* Schedule 1 : r1(K) -> w1(K) -> r1(H) -> w1(H) -> c1 -> r2(H) -> w2(H) -> c2
* Schedule 2 : r2(H) -> w2(H) -> c2 -> r1(K) -> w1(K) -> r1(H) -> w1(H) -> c1

트랜잭션들(Operations)이 겹치지 않고 순차적으로 실행되는 것을 Serial Schedule 이라고 한다.  

**특징**   
* 한번에 하나의 transaction만 실행되기에 좋은 성능을 낼 수 없고 현실적으로 사용할 수 없는 방식이다.  

### Nonserial schedule

* Schedule 3 : r1(k) -> w1(k) -> r2(H) -> w2(H) -> c2 -> r1(H) -> w1(H) -> c1
* Schedule 4 : r1(k) -> w1(k) -> r1(H) -> r2(H) -> w2(H) -> c2 -> w1(H) -> c1

트랜잭션들(Operations)이 겹쳐서(ineterleaving) 실행되는 것을 Nonserial schedule 이라고 한다.   

**특징**   
* transaction 들이 겹쳐서 실행되기에 동시성이 높아져서 같은 시간동안 더많은 transaction 들을 처리할 수 있다.  
* 단, transaction 들이 어떤 형태로 겹쳐서 실행되는지에 따라 데이터 정합성에 문제가 생길 수 있다.  



## serializability
