# MVCC(multiversion concurrenct control)

**MVCC**
* 데이터를 읽을 때 **특정 시점** 기준으로 가장 최근에 **커밋**된 데이터를 읽는다.   
    * Isolation level 에 따라 다르다.  
    * MySQL 에서는 이러한 동작을 Consistent Read 라고 부른다.     
* 데이터 변화 이력을 관리한다.(RDBMS가 변화 이력을 내부적으로 관리한다 == 추가적인 저장 필요)    
* **read와 wrtie는 서로를 block 하지 않는다.**       
      
현재 시중에서 사용되는 대부분의 RDBMS는 MVCC 를 기반으로 설계되어 있다.       
Isolation 4 단계중 ReadCommitted 와 Repeatable Read 는 MVCC를 기반으로 구현되고 있으며     
Serializable 과 Read UnCommitted 는 아래와 같이 조금 다르게 동작한다.      

**Serializable**
* MySQL : MVCC로 동작하기 보다는 lock 으로 동작한다.  
* PostgreSQL : SSI(Serializable snapshot isolation) 기법이 적용된 MVCC로 동작 

**Read UnCommited**
* MVCC는 커밋된 데이터를 읽기 때문에, 이 Level 에서는 보통 MVCC가 적용되지 않는다. 
* 참고로, PostgreSQL 은 해당 레벨이 존재하지만, Read Committed 처럼 동작한다.  

## PostgreSQL 에서의 Lost Update
* TransactionA : Read Committed
* TransactionB : Read Committed 
* MVCC Rule : Write == Protected Area write

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 10
Database -> TransactionA : read(x) => 50
TransactionA -> Database: write(x = 10)

TransactionB -> Database : read(x) => 50
TransactionB -> Database : wrtie wait(x = 80) 

Database -> TransactionA : read(x) => 10
TransactionA -> Database  : write(y = 50) 
TransactionA -> Database : commit

TransactionB -> Database : wrtie start(x = 80) 
TransactionB -> Database : commit

Database -> Database : x = 80, y = 50
```

* MVCC 기법으로 인하여 실제 write protectedArea 에 저장된다.
* 즉, 커밋이 바로 일어나는 것이 아니므로, 다른 트랜잭션은 write 작업에 대해 Lock 이 걸린다.
	* 단, MVCC 는 Write-Write 에 대해서만 Lock 을 잡는다는 점을 기억하자 

이와 같이 두개 이상의 트랜잭션에서,
한 트랜잭션에서 발생한 update 기록이 누락되는 현상을 LostUpdate 라고 부르며
실제 실무에서도 가장 많이 발생하는 이슈이기도 하다.  


### PostgreSQL 에서의 Lost Update 해결방안 1단계
* TransactionA : Read Committed
* TransactionB : Repeatable Read
* MVCC Rule : Write == Protected Area write

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 10
Database -> TransactionA : read(x) => 50
TransactionA -> Database: write(x = 10)

TransactionB -> Database : read(x) => 50
TransactionB -> Database : wrtie wait(x = 80) 

Database -> TransactionA : read(x) => 10
TransactionA -> Database  : write(y = 50) 
TransactionA -> Database : commit

TransactionB -> Database : ~~wrtie cancle(x = 80)~~
TransactionB -> Database : rollback

Database -> Database : x = 10, y = 50
```
* PostgreSQL 에서의 Repeatable Read 는 아래와 같은 특징을 가진다.
	* 같은 데이터에 먼저 다른 transaction의 commit(write)이 있다면
	  나중 transaction 은 rollback 된다. 
* 두 트랜잭션이 목표하는 작업을 모두 성공하지는 않았지만
  하나의 트랜잭션이 rollback 하면서 lost update를 막았다.(롤백은 의도된 실행이다.)
* 이와 같은 작업을 first-updater-win 이라고 부른다.  

### PostgreSQL 에서의 Lost Update 해결방안 2단계
* 1단계는 TransactionA이 먼저 실행되었다고 가정한 케이스다.
* 그 반대의 경우에 대해서도 방어 로직을 작성해주어야한다.  

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 10
Database -> TransactionB : read(x) => 50
Database -> TransactionA : read(x) => 50

TransactionB -> Database : write(x = 80)
TransactionA -> Database : write wait(x = 10) 
TransactionB -> Database : commit

TransactionA -> Database : **write cancle(x = 10)** 
TransactionA -> Database : rollback

Database -> Database : x = 80, y = 10
```

## MySQL 에서의 Lost Update
* TransactionA : Repeatable Read
* TransactionB : Repeatable Read
* MVCC Rule : Write == Protected Area write

MySQL은 PostgreSQL 과 달리 first-updater-win 방식을 지원하지 않는다.
그렇기에 두 트랜잭션 모두 Repeatable Read 로 잡아도 아래와 같은 결과가 나온다. 

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 10
Database -> TransactionB : read(x) => 50
Database -> TransactionA : read(x) => 50

TransactionB -> Database : write(x = 80)
TransactionA -> Database : write wait(x = 10) 
TransactionB -> Database : commit

TransactionA -> Database : write start(x = 10)
Database -> TransactionA : read(y) => 10
TransactionA -> Database : write(y = 50)
TransactionA -> Database : commit

Database -> Database : x = 80, y = 50
```

### MySQL 에서의 Lost Update 해결방안
* MySQL 의 Repeatable Read 문제를 해결하기 위해서는 Locking Read 를 활용한다.
* `SELECT` 쿼리 뒤에 `FOR UPDATE` or `FOR SHARE` 를 붙여서 사용하면 된다.  
* Locking Read는 Read 를 하면서 조회 데이터에 대한 Write Lock 을 취득을 할 수 있는 기법이다. 


```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 10
Database -> TransactionB : read(x) => 50 🔐
Database -> TransactionA : read wait(x) => 50 🔐

TransactionB -> Database : write(x = 80) 
TransactionB -> Database : commit

Database -> TransactionA : read start(x) => 80
TransactionA -> Database : write(x = 40) 
Database -> TransactionA : read(y) => 10 🔐
TransactionA -> Database : write(y = 50)
TransactionA -> Database : commit

Database -> Database : x = 40, y = 50
```
* read start(x) 에 대해서 값이 50으로 나올 수 있을것이라 생각했을 수도 있다.(repeatable read니까)
* Lock Read 는 가장 최근의 commit 된 데이터를 읽는다.
* 그렇기에 업데이트된 80 값을 읽어오고 이후 정상적인 동작을 수행한다.  

```
FOR UPDATE : Exclusive Lock 획득 
FOR SHARE : Read Lock 획득 

다른 RDBMS에도 존재하나 동작 방식은 벤더마다 다르다.
```


## RepeatableRead 에서 발생하는 Lost Update 다른 케이스

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 10, y = 10

Database -> TransactionA : read(x) => 10 
Database -> TransactionB : read(x) => 10 
Database -> TransactionA : read(y) => 10 
Database -> TransactionB : read(y) => 10 

TransactionA -> Database : write(x = 20)
TransactionB -> Database : write(y = 20)

TransactionA -> Database : commit
TransactionB -> Database : commit

Database -> Database : x = 20, y = 20
```

* write skew 현상은 MVCC 에서 동일하게 발생할 수 있다.
* 이는 MySQL 이랑 PostgreSQL 에서 모두 발생한다.(이전 케이스는 MySQL 에서만 발생)  

이를 해결하기 위해서는 Locking Read 를 활용한다. 

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 10, y = 10

Database -> TransactionA : read(x) => 10 🔐
Database -> TransactionB : read wait(x) 🔐 
Database -> TransactionA : read(y) => 10 🔐
TransactionA -> Database : write(x = 20)
TransactionA -> Database : commit

Database -> TransactionB : read start(x) => 20 🔐
Database -> TransactionB : read(y) => 10 🔐
TransactionB -> Database : write(y = 30)
TransactionB -> Database : commit

Database -> Database : x = 20, y = 30
```

* 위 그림은 MySQL 기반으로 해결된 그래프다.
* Locking Read 로 인하여, 후순위 트랜잭션의 read는 블락이 되었고
* 이후 작업을 진행했을 때, 최신 데이터를 반영오게 되어 동시성 문제가 해결되었다.  

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 10, y = 10

Database -> TransactionA : read(x) => 10 🔐
Database -> TransactionB : read wait(x) 🔐 
Database -> TransactionA : read(y) => 10 🔐
TransactionA -> Database : write(x = 20)
TransactionA -> Database : commit

Database -> TransactionB : ~~read cancle(x) => 20 🔐~~
TransactionB -> Database : rollback

Database -> Database : x = 20, y = 10
```

* 위 그림은 PostgreSQL 기반으로 해결된 그래프다.
* PostgreSQL 은 first-updater-win 기반이기에
  다른 트랜잭션에서 x가 업데이트 되었으니 롤백을 진행한다.  

# 이외에도, Serializable 방식 사용
* MySQL :
	* repeatable read 와 유사한 방식이다.
	* tx의 모든 평범한 select 문은 암묵적으로 select for share 처럼 동작한다
	* 즉, MVCC 가 아닌 Lock 기반으로 동작을 한다.
	* share 인 이유는, 쓰기락의 경우 성능에 이슈가 있어서 채택했다.(단 데드락 발생률은 높다)

* PostgreSQL
	* SSI 로 구현되어 있다.
	* first-committer-winner 방식을 사용하고 있다. 
