# Isolation Level
## DirtyRead
* Transaction A : x와 y를 더한 값을 y에 저장한다.
* Transaction B : y에 50을 더한다. 

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 10, y = 20

Database -> TransactionA : read(x) => 10
TransactionB -> Database : write(y = 70)

Database -> TransactionA : read(y) => 70
TransactionA -> Database : write(y = 80)
TransactionA -> Database : commit

TransactionB -> Database : abort : rollback(y = 20)

Database -> Database : Result : x = 80, y = 20
```
* Commit 되지 않은 변화를 읽음으로써
  발생하는 데이터 정합성 문제
* 위 예시에서는 롤백된 데이터를 기반으로 commit 을 했다.

## NonRepeatable Read(Fuzzy Read)
* Transaction A : x를 두번 읽는다.
* Transaction B : x에 40을 더한다. 

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 10

Database -> TransactionA : read(x) => 10
Database -> TransactionB : read(x) => 10
TransactionB -> Database : write(x = 50)
TransactionB -> Database : commit

Database -> TransactionA : read(x) => 50
TransactionA -> Database : commit
Database -> Database : Result : x = 50
```

* 같은 값을 한 트랜잭션에서 2번 읽었음에도 다른 값이 조회되는 문제 발생  
* read 뿐 아니라 write 작업도 있었다면 매우 큰 문제를 초래하게 될 수 있다.
* NonRepeatable Read(Fuzzy Read) 는 같은 데이터의 값이 달라짐을 의미한다.

## Phantom Read
* Transaction A : v가 10인 데이터를 두번 읽는다. 
* Transaction B : t2의 v2를 10으로 바꾼다. 

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : t1(..., v = 10),\nt2(..., v = 50)

Database -> TransactionA : read(v = 10) => t1
TransactionB -> Database : write(t2.v = 10)
TransactionB -> Database : commit

Database -> TransactionA : read(v = 10) => t1, t2
TransactionA -> Database : commit

Database -> Database : Result : t1(..., v = 10),\nt2(..., v = 10)
```
* Phantom read : 없던 데이터가 갑자기 생기는 현상

위와 같은 3가지 문제사항을 발생하지 않게 만들 수 있다.     
그러나 제약사항이 많아지면서 동시 처리 가능한 트랜잭션 수가 줄어들어     
결국 DB의 전체 처리량이 하락하게 된다.    

이를 해결하기 위해서   
일부 이상한 현상을 허용하는 몇가지 level 을 만들어서
사용자가 필요에 따라 적절하게 선택할 수 있도록하자고 나온게 Isolation 4 level 이다. 

| Isolation Level  | Dirt read | Non-repeatable read | Phantom read |
| ---------------- | --------- | ------------------- | ------------ |
| Read uncommitted | O         | O                   | O            |
| Read committed   | X         | O                   | O            |
| Reapeatable read | X         | X                   | O            |
| Serializable     | X         | X                   | X            |

정리하자면, 
3가지 이상 현상을 정의하고 어떤 현상을 허용하는지에 따라서 Isolation level 이 구분된다.  
애플리케이션 설계자는 Isolation level 을 통해 
전체 처리량과 데이터 일관성 사이에서 어느정도 거래를 할 수 있다.  

이 같은 내용은 1992년 11월에 발표된 SQL 표준에서 정의된 내용이다. 
그런데 ANSI/ISO standard SQL 92 에서 정의한 isolation level 을 비판하는 논문이 등장한다. 

1. 세가지 이상 현상의 정의가 모호하다.
2. 이상 현상은 세가지 외에도 더 있다.
3. 상업적인 DBMS 에서 사용하는 방법을 반영해서, isolation level 을 구분하지 않았다.

## Dirty Write 
* TransactionA : x를 10으로 바꾼다.
* TransactionB : x를 100으로 바꾼다.

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 0

TransactionA -> Database : write(x = 10)
TransactionB -> Database : write(x = 100)

TransactionA -> Database : abort -> 100으로 돌림
Database -> Database : x = 100
```
* Dirty Write : commit 이 안 된 데이터를 write 할때 발생하는 문제 
* rollback 시 정상적인 recovery가 되는 것은 매우 중요하기에
  모든 isolation level 에서는 dirty write 를 허용해서는 안 된다. 

## Lost Update
* TransactionA : x에 50을 더한다.  
* TransactionB : x에 150으로 더한다.

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 200

TransactionA -> Database : read(x) => 50
TransactionB -> Database : read(x) => 50
TransactionB -> Database : write(x = 200)
TransactionB -> Database : commit

TransactionA -> Database : write(x = 100)
TransactionA -> Database : commit
Database -> Database : x = 100
```
* 다른 트랜잭션이 write 한 값을
  이후에 동작한 트랜잭션이 값을 덮어쓰는 형태가 나타나고 있다.  
* Serial 하게 실행이 되었다면, 기존 값들이 차례대로 누적되어야 하는데 그러지 못한 것이다.
* 이같은 update 가 반영이 되지 않는 형상을 lost update 라고 부른다.

## Dirty Read 확장판
* dirty read 는 commit 되지 않은 데이터가 롤백되면서 발생하는 문제를 말한다.
* 비판된 논문에서는 롤백이 아니여도 문제가 발샐한다고 이야기하고 있다.

**상황**
* TransactionA : x가 y에 40을 이체한다.  
* TransactionB : x와 y를 읽는다. 

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 50

Database -> TransactionA : read(x) => 50
TransactionA -> Database : write(x = 10)

Database -> TransactionB : read(x) => 10
Database -> TransactionB : read(y) => 50
TransactionB -> Database : commit

Database -> TransactionA : read(y) => 50
TransactionA -> Database : write(y = 90)
TransactionA -> Database : commit

Database -> Database : x = 10, y = 90
```

* 최종 결과 데이터는 문제가 없지만, TransactionB의 읽기에 문제가 있다.
* 총 데이터의 합은 100이지만, TransactionB가 읽은 데이터의 합은 60으로 정합성이 맞지 않다.
* 논문에서는 위와 같은 상황에 대해서도 
  commit 되지 않은 데이터를 읽을 때 발생하므로 dirty read라 주장한다.
  즉, abort가 발생하지 않아도 dirty read가 될 수 있다. (기존 논문은 롤백되었을 때만 dirty read라 표현)

## Read Skew
**상황**
* TransactionA : x가 y에 40을 이체한다.  
* TransactionB : x와 y를 읽는다. 

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 50

Database -> TransactionB : read(x) => 50

Database -> TransactionA : read(x) => 50
TransactionA -> Database : write(x = 10)
Database -> TransactionA : read(y) => 50
TransactionA -> Database : write(y = 90)
TransactionA -> Database : commit

Database -> TransactionB : read(y) => 90
TransactionB -> Database : commit

Database -> Database : x = 10, y = 90
```

* 위의 경우도, TransactionB 가 읽은 데이터의 총합은 140으로 정합성이 맞지 않다.  
* 이 같이 inconsistent 한 데이터를 읽기를 Read Skew 라고 부른다. 
* NonRepeatable Read 와 비슷하지만, 
  여기서는 x 만이 아닌, x와 관련된 y 에 대해서도 작업을 하므로 Read Skew 라 부른다.  

## Write Skew
**상황**
* DB에 존재하는 x와 y의 합이 0 이상이어야 한다.   
* TransactionA : x에서 80을 인출한다.  
* TransactionA : y에서 90을 인출한다.  

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 50

Database -> TransactionA : read(x) => 50
Database -> TransactionA : read(y) => 50

Database -> TransactionB : read(x) => 50
Database -> TransactionB : read(y) => 50


TransactionA -> Database : write(x = -30)
TransactionB -> Database : write(y = -40)
TransactionA -> Database : commit
TransactionB -> Database : commit

Database -> Database : x = -30, y = -40
```
* 위의 경우는, 각각의 트랜잭션이 실행하면서 값을 감소시키는데 조건에 따른 정합성이 맞지 않는 케이스다.
* 이 같이 inconsistent 한 데이터를 쓰기를 Write Skew 라고 부른다. 

## Phantom Read 확장 
**상황**
* DB에는 튜플 하나 있음
	* t1(..., v = 7)
	* cnt : v > 10 를 만족하는 튜플의 개수
* TransactionA : v > 10 데이터와 cnt 를 읽는다. 
* TransactionA : v = 15인 t2를 추가하고 cnt를 1증가한다. 

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : t1(..., v = 7), cnt = 0

Database -> TransactionA : read(v > 10) => 없음

TransactionB -> Database : write(insert(t2 : ..., v = 15))
Database -> TransactionB : read(cnt) => 0
TransactionB -> Database : write(cnt = 1)
TransactionB -> Database : commit

Database -> TransactionA : read(cnt) => 1
TransactionA -> Database : commit
```

* 중간에 저장된 값으로 인해, TransactionA가 읽은 연관된 두 데이터 사이의 정합성은 맞지 않는다.
* 같은 데이터 2번 읽는 경우가 아니더라더, 
  연관이 있는 데이터를 읽었을 때 데이터 정합성이 발생할 수 있다.
* 이 같은 경우도 포함해서 Phantom Read 로 인식해야한다.  

## SnapShot Isolation
* 상업적인 DBMS 에서 사용되는 방법을 반영해서 Isolation level 을 구분한 isolation 
* 기준 표준에서 정의한 isolation 들은 이상한 현상 3가지를 정의하고 얼마만큼 허용할 것이냐에 초점
  snapshot isolation 은 conccurency control 이 어떻게 동작할지 그 구현을 바탕으로 정의됨 초점


**상황**
* TransactionA : x가 y에 40을 이체 
* TransactionA : y에 100을 입금

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 50, y = 50
Database -> TransactionA : read(x) => 50 (스냅샷 저장)
TransactionA -> Database : write(x = 10) (스냅샷 저장)

Database -> TransactionB : read(y) => 50 (스냅샷 저장)
TransactionB -> Database : write(y = 150) (스냅샷 저장)
TransactionB -> Database : commit

Database -> TransactionA : read(y) => 50
TransactionA -> Database : write(y = 90)
TransactionA -> Database : abort (write x write conflict)
```
* 동일한 데이터에 대한 `write x write confilct` 이 발생하면
  먼저 커밋된 트랜잭션만 인정하는 방식이다.(이후 write 트랜잭션은 롤백된다.)
* transaction 시작 전에 commit 된 데이터만 보인다.
* first-committer-win 기반으로 동작한다고 보면 된다.  
* MVCC 의 한 종류이다.  

