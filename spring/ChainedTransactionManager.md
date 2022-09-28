# ChainedTransactionManager
    
모놀리틱 아키텍처에서, 상황에 따라 여러 DB 에 대한 트랜잭션을 보장해줘야 하는 경우도 있다.      
`ChainedTransactionManager`는 말 그대로 여러개의 트랜잭션 매니저를 하나로 묶어(Chain)사용하는 방식을 지원해주며         
트랜잭션의 시작과 끝에서 연결된 트랜잭션들을 순차로 Start/Commit 시킴으로써 하나의 트랜잭션으로 실행되는것 처럼 동작하게 한다.    

![1](https://user-images.githubusercontent.com/50267433/192659315-1f6c45d6-155f-44b9-9520-4c6ec019c33d.png)

단, `ChainedTransactionManager`이라도 완벽한 트랜잭션을 지원해주지는 않는다.  

정상적인 케이스는 아래와 같다.  
* Tx1 (Start) -> Tx2 (Start) -> Logic -> Tx2 (COMMIT) -> Tx1 (COMMIT) 
* Tx1 (Start) -> Tx2 (Start) -> Logic -> Tx2 (ROLLBACK) -> Tx1 (ROLLBACK) 

반면 비정상적인 케이스는 아래와 같다.  
* Tx1 (Start) -> Tx2 (Start) -> Logic -> Tx2 (COMMIT) -> Tx1 (ROLLBACK)   
  
commit 을 진행하면 롤백이 불가능하다.        
그렇기에 `ChainedTransactionManager` 을 사용하고자 한다면 트랜잭션 순서를 잘 제어해야한다.      
(에러가 날 확률이 높은 트랜잭션을 후순위 chain으로 묶어주셔야 조금 더 안전한 트랜잭션을 구성 할 수 있다.)   

![3](https://user-images.githubusercontent.com/50267433/192659590-a56e1330-7424-45f5-8744-06be5dedca50.png)

위 그림과 같이 커밋 작업이 일렬 밀려있는 형태면 좋다.   

# JtaTransactionManager
> [JtaTransactionManager](https://docs.spring.io/spring-boot/docs/2.1.6.RELEASE/reference/html/boot-features-jta.html)




# TransactionSynchronization
