# Recoverability

**상황 가정**
1. K 라는 사람이 존재하며, 100만원을 보유하고 있다.
2. H 라는 사람이 존재한다. 200만원을 보유하고 있다.
3. 작업
	1. K가 H에게 20만원을 이체한다.
	2. H는 본인 계좌에 30만원을 입금한다.

![[Pasted image 20221002121652.png]]
* 각각의 트랜잭션이 정상 동작을 하다가
* schedule 내에서 커밋된 트랜잭션이 
  롤백된 트랜잭션의 write 한 데이터를 읽었을 경우
  unrecoverable schedule 이라고 이야기한다.  
* 위 예시로 들면, 
  트랜잭션 2 부분이 abort 되었고 
  트랜잭션 1 부분이 commit 되었는데
  실상, abort 된 데이터를 기준으로 commit 이 된 거다. 
* 롤백을 해도 이전 상태로 회복 불가능할 수 있기에,
  이런 스케줄은 DBMS가 허용하면 안된다.  


unrecoverable schedule 이 아니라 
recoverable schedule 을 사용해야한다.  
그려면 어떤 스케줄이 recoverable schedule 일까?  

![[Pasted image 20221002122134.png]]
![[Pasted image 20221002122208.png]]

* 핵심은, 의존성이 있으면 의존성이 있는 트랜잭션이 종료될때 까지 의존하는 트랜잭션이 커밋 하면 안되게 한다.
* schedule 내에서 그 어떤 transaction 도 자신이 읽은 데이터를 write한 transaction 이 먼저 commit/rollback 전까지는
  commit 하지 않는 경우의 스케줄을 recoverable schedule 이라한다.
* rollback 할때 이전 상태로 온전히 돌아갈 수 있기에
  DBMS는 이러한 schedule 만 허용해야한다.  

**cascading rollback**
* 하나의 트랜잭션이 롤백하면, 
  의존성이 있는 다른 트랜잭션도 롤백해야한다.
* 단, 여러 트랜잭션의 롤백이 연쇄적으로 일어나면 처리하는 비용이 많이 든다.  
* 이를 해결하기 위해서, write한 트랜잭션이 commit/rollback 한뒤에 데이터를 읽는 스케줄만을 허용하자 

![[Pasted image 20221002122654.png]]
* 커밋이 되면 두 트랜잭션 모두 커밋이 되는 형태이며 

![[Pasted image 20221002122735.png]]
* 커밋이 안되고 롤백되어도, 
  다른 트랜잭션은 자신의 작업을 그대로 수행하면 된다.

**cascadeless schedule(avoid cascading rollback)**
* schedule 내에서 어떤 transaction 도 commit 되지 않은
  transaction 들이 write 한 데이터를 읽지 않는 경우를 의미한다.

그러나 **cascadeless schedule** 또한 완벽하지는 않다. 


**상황 가정**
1. H 사장이 3만원이던 피자가격을 2만원으로 낮추려는데
2. K 직원도 동일한 피자 가격을 실수로 1만원으로 낮추려했을 때 

![[Pasted image 20221002123437.png]]

* 위 스케줄도 cascadeless schdule 이라 볼 수 있다.
* schedule 내에서 어떤 트랜잭션도 
  commit 되지 않은 transaction들이 write 한 데이터는 
  읽지 않는 경우 이기 때문이다. 
* 이 문제를 해결하려면,
  schedule 내에서 어떤 트랜잭션도 
  commit 되지 않은 transaction들이 write 한 데이터는 
  쓰지도 읽지도 않는 경우 이기 때문으로 봐야한다.  
* 이러한 스케줄을 strict schedule 이라고 말한다.
* strict schedule 은, 롤백할 때 리커버리가 쉽다.
  트랜잭션 이전 상태로 돌려놓기만 하면 되기 때문이다.  

![[Pasted image 20221002123754.png]]

* concurrency control : serializability + recoverability
* 이와 관련된 트랜잭션의 속성이 isolation 이라고 보면 된다.
