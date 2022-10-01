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

## PostgreSQL 에서의 Lost Updae


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


TransactionB -> Database : wrtie start(x = 80) 
```


* hello

