# 개요

회사에서 담당하는 코어 프로젝트를 JDK11 로 마이그레이션하는 과정에서 이슈가 하나 발생했습니다.      
`ChainedTransactionManager` 이 SpringBoot 2.5 부터 Deprecated 되면서 대체제를 찾아야했던겁니다.   

우선, 프로젝트에서 왜 `ChainedTransactionManager`을 사용했는지를 알아야겠다는 생각이 들었고  
여기서 학습된 개념을 통해서 어떻게 개선해야할지에 대해서 간략히 짚고 넘어가보려 합니다.    

## ChainedTransactionManager

단일 프로젝트를 기준으로, 여러 DB 에 대한 트랜잭션을 보장해줘야 하는 경우도 있습니다.
이때 사용하는 것이 ChainedTransactionManager  로서
말 그대로 여러개의 트랜잭션 매니저를 하나로 묶어(Chain)사용하는 방식을 지원해줍니다.

이에 대해서 정상적인 케이스와 비정상적인  케이스를 나타내면 아래와 같습니다.

**정상적인 케이스**
* Tx1 (Start) -> Tx2 (Start) -> Logic -> Tx2 (COMMIT) -> Tx1 (COMMIT)
* Tx1 (Start) -> Tx2 (Start) -> Logic -> Tx2 (ROLLBACK) -> Tx1 (ROLLBACK)

**비정상적인 케이스**
* Tx1 (Start) -> Tx2 (Start) -> Logic -> Tx2 (COMMIT) -> Tx1 (ROLLBACK)

DB는 Commit 을 진행할 경우, 롤백 하지 못한다는 특성을 가지고 있습니다(SnapShot 방식 제외)
이 같은 문제로 인하여 Spring 팀에서는 boot 2.5 부터 Deprecated 를 선언했으며
현 2.7.x 그리고 더 나아가 3.0.x 에서는 사용조차 힘들어지리라 판단이 됩니다.

참고 자료 1
참고 자료 2

## 대안 1. TransactionSynchronization

ChainedTransactionManager 의 주석을 보면 아래와 같습니다.



정리하자면, TransactionSynchronization 을 사용하는 것을 추천한다고 정리되어있습니다.
TransactionSynchronization 의 코드는 다음과 같습니다.

```java
public interface TransactionSynchronization {  

   int STATUS_COMMITTED = 0;  
   int STATUS_ROLLED_BACK = 1;  
   int STATUS_UNKNOWN = 2;  
  
   default Mono<Void> suspend() {  
      return Mono.empty();  
   }  
  
   default Mono<Void> resume() {  
      return Mono.empty();  
   }  
  
   default Mono<Void> beforeCommit(boolean readOnly) {  
      return Mono.empty();  
   }  
  
   default Mono<Void> beforeCompletion() {  
      return Mono.empty();  
   }  
  
   default Mono<Void> afterCommit() {  
      return Mono.empty();  
   }  
  
   default Mono<Void> afterCompletion(int status) {  
      return Mono.empty();  
   }  
  
}
```

다만, TransactionSynchronization 에도 문제가 있는데
TransactionSynchronization 는 인터페이스므로 이에 해당하는 구현체를 만들어줘야 하며
각각의 케이스마다 `afterCommit()` , `beforeCommit()` 에 맞는 코드를 작성해야한다는 점이 있습니다.

```java
@Service
public class OneService {

    @Autowired
    OneDao dao;

    @Transactional
    public void a transactionalMethod() {
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronizationAdapter(){
		        public void afterCommit(){
	                System.out.println("commit!!!");
	            }
	        }
       );
	    dao.save();
    }
}
```

(구현체를 만들수도 있지만, 각 케이스마다 구현체를 만들어줘야 다양한 체이닝이 가능합니다.)
이러한 문제를 비교적 해소하고자, Event 방식으로 전환을 고려할 수도 있습니다만
이 같은 경우도, 분산된 Transaction 에 대해 보상 트랜잭션을 만들어서 관리해야하는 문제가 있습니다

-> 1차 트랜잭션 성공
-> 1차 트랜잭션 성공 Event 발행
-> Listen
-> 2차 트랜잭션 실패
-> 2차 트랜잭션 실패 Event 발행
-> Listen
-> delete/update/drop

대안 2. JtaTransactionManager

분산 트랜잭션과 XA, JTA

분산 트랜잭션

분산 트랜잭션(distributed transaction)은 2개 이상의 네트워크 시스템 간의 트랜잭션을 의미합니다.
일반적으로 시스템은 트랜잭션 리소스의 역할을 하고,
트랜잭션 매니저는 리소스에 관련된 모든 동작에 대해 트랜잭션의 생성 및 관리를 담당합니다.

XA

XA는 분산 트랜잭션 처리를 위해 X/Open이 제정한 표준 스펙입니다.
멀티 트랜잭션 관리자와 로컬 리소스 관리자 사이의 인터페이스,
리소스 관리자가 트랜잭션을 처리하기 위해 필요한 것을 규정하고 있습니다.

Two Phase Commit 수행을 통해,
분산된 데이터베이스에서 발생하는 각 트랜잭션을 원자적인 트랜잭션으로 구성할 수 있게 해줍니다.

Two Phase Commit
* 여러 노드에 거쳐서 원자성 트랜잭션 커밋을 달성하기 위한 알고리즘
* 분산 데이터베이스의 트랜잭션 처리를 위해서 사용하는 고전적인 방법
* 코디네이터(트랜잭션 관리자)를 사용하여, DB 들에게 커밋 여부를 확인하고 커밋하는 방법
* 단 하나의 DB 노드에서라도 커밋이 될 수 없다면 롤백 진행(구현마다 다르지만) 
* 블록킹 방식을 따르기에 비교적 느려지는 현상 발생

JTA

JTA(Java 트랜잭션 API)는 XA 리소스간의 분산 트랜잭션을 처리하는 Java API 입니다.
JTA API는 javax.transaction와 javax.transaction.xa 두 개의 패키지로 구성되며
JTA 인터페이스를 구현한 오픈소스에는 다음과 같은 프로젝트들이 있습니다.

Atomikos

Bitronix

Narayana

JtaTransactionManager

JTA 인터페이스를 기반으로 만든 TransactionManager 로 스프링에서 공식 지원하고 있습니다.



위 그림에서 설명하듯 각 레이어의 인터페이스에 해당하는 구현체를
분산 트랜잭션을 처리할 수 있는 구현체로 교체하면 됩니다.

implementation 'org.springframework.boot:spring-boot-starter-jta-atomikos'
implementation 'org.springframework.boot:spring-boot-starter-jta-bitronix'
implementation 'org.springframework.boot:spring-boot-starter-jta-narayana'

더 나아가야할 점

팀업에서는 애플리케이션단에서 RO/RW 를 분리하여 DB의 부하 분산을 진행합니다.
스프링에서 RO/RW 를 분리하는 방법에는 크게 2가지가 존재합니다.

RO/RW 별로 데이터소스를 만들어 개발자가 이를 인지하며 개발

단일 데이터 소스에서 트랜잭션의 분기처리를 Lazy 하게 처리

1. RO/ RW 별로 데이터소스를 만들어 개발자가 이를 인지하며 개발



RW/RO 를 분리한 Repository를 이용하여
개발시마다 인지하여 알맞은 Repository를 명시적으로 선언하여 개발을 해야합니다.

단점
1. 다른 노드 Undo 영역 참조 불가로 인한 문제 
    위 예시와 같이 Read-Write 노드에서 유저를 새로 생성하고,
    Read-Only 노드에서 전체 유저의 개수를 가져오게 된다면?
    Undo 영역은 노드간에 공유되지 않기 때문에 
    count는 커밋되지 않은 신규 유저 값을 제외하고 카운트하게 됩니다.
2. 명시적으로 RW/RO Repository 를 지정하는 방식   
    레플리카를 여럿 두는 확장된 작업을 진행해야할 경우
    HAProxy 와 같은 DB 로드밸런싱에 의존해야한다.
3. 샤딩을 도입하게 된다면 HAProxy 의 역할이 애매해지기에  

2. 단일 데이터 소스에서 트랜잭션의 분기처리를 Lazy 하게 처리(작성중)
