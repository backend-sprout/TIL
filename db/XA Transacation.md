# XATransaction
# 분산 트랜잭션
글로벌 트랜잭션이라고도 불리며, 
여러개로 분산된 리소스들 각각에 대한 트랜잭션들을 하나의 트랜잭션으로 묶은 것을 의미한다.
하나의 리소스가 실패하면 전체를 RollBack 하는 특징을 가지고 있다. 

Distributed Transaction Processing(DTP) 아키텍처는
여러 Application 이 TransactionManager 를 이용하여   
여러 다른 Resource Manage가 제공하는 리소스를 공유할 수 있게 해주는 
**표준 아키텍처 또는 인터페이스를 의미한다.** 

# XA
XA는 2P(two phase commit) 을 통한, 분산 트랜잭션 처리를 위한 X/Open 에서 명시한 표준이다.  
XA는 분산트랜잭션 환경에서 Transaction Manager 와 Resource manage 사이에 
통신을 담당하는 하나의 표준화된 인터페이스를 의미한다.   
각각의 벤더사별로 해당 인터페이스에 맞는 동작을 구현해서 제공하고 있다.   

글로벌 트랜잭션을 사용하는 응용 프로그램들은 
하나 혹은 그 이상의 Resource Manager 와 Transaction Mananger 를 포함한다.  

**Resource Manager**
일반적으로 컴퓨터 혹은 서버의 공유 리소스를 관리하며 트랜잭션 리소스들에 대한 접근을 제공한다.
* Shared Resource : Database 
* Reseource Manager : DBMS

**Transaction Manager**
글로벌 트랜잭션의 부분인 Transaction Resource 들을 통합(coordination) 한다.   
각 Resource Manager 별로 XID 를 생성하여 transaction 진행을 관리하고   
전체 Resource Manager 들의 트랜잭션을 commit 하고나 rollback 하는 기능을 수행한다.   
Transaction Manager 는 각각의 트랜잭션을 다루는 Resource Manager와 함께 정보를 주고 받는다.

**Transaction Branches**
글로벌 트랜잭션이 포함하는 개별적인 로컬 트랜잭션을 Transaction Branches 라고 부른다.   
리소스가 DB인 경우 각 branch 는 DBMS 내부의 로컬 트랜잭션을 의미하며, 
글로벌 트랜잭션과 그 브런치들은 naming schema에 의해 구별된다.  

**2 phase commit**
글로벌 트랜잭션을 실행하기 위해서는 2PC(2 phase commit)를 사용한다.
2PC(2 phase commit)는 글로벌 트랜잭션의 branch들에 의해 수행되는 action들 뒤에 발생한다.  

-   1st Phase :
  모든 트랜잭션 브런치들이 준비가 되는 단계로, 
  각 트랜잭션 매니저가 데이터베이스 노드에 commit을 하기 위한 
  prepare 메시지를 보내 commit을 준비하는 단계
-   2nd Phase :
  트랜잭션 매니저가 참여한 모든 데이터베이스 노드로 부터 
  prepare 완료 메시지를 받을 때까지 대기하며, 
  prepare 메시지 중 하나라도 OK가 아니라면 rollback, 모두 OK라면 commit 메시지를 보낸다.


# MariaDB에서 XA 구현 
XA Transaction은 Transaction Manager(애플리케이션)가   
여러 리소스를 포함하는 트랜잭션을 제어하는 **분산 트랜잭션**을 허용하도록 설계되어있다.  

분산 트랜잭션을 허용한다는 것은 
여러 개의 분리된 트랜잭션 리소스들이 하나의 글로벌(global) 트랜잭션에 포함되어 
**하나의 그룹으로 트랜잭션 동작을 하는 것을 의미한다.**    

MariaDB 에서의 XA Transaction은 이를 지원하는 '스토리지 엔진'에서만 사용할 수 있다. 
적어도 InnoDB, TokuDB, SPIDER 및 MyRocks가 지원한다.    

InnoDB의 경우, 
innodb_support_xa 서버 시스템 변수를 0으로 설정하여 
XA Transaction 을 비활성화 할 수 있다. 

## MARIADB 엔진 조회
MariaDB에서는 쿼리 조회를 통해 스토리지 엔진에 따라 XA 지원여부를 확인할수 있다.

```
MariaDB [(none)]> SELECT engine, support, transactions, xa FROM information_schema.engines;
+--------------------+---------+--------------+------+
| engine             | support | transactions | xa   |
+--------------------+---------+--------------+------+
| SPIDER             | YES     | YES          | NO   |
| MRG_MyISAM         | YES     | NO           | NO   |
| MEMORY             | YES     | NO           | NO   |
| Aria               | YES     | NO           | NO   |
| MyISAM             | YES     | NO           | NO   |
| SEQUENCE           | YES     | YES          | NO   |
| InnoDB             | DEFAULT | YES          | YES  |
| PERFORMANCE_SCHEMA | YES     | NO           | NO   |
| CSV                | YES     | NO           | NO   |
+--------------------+---------+--------------+------+
```

XA Transaction 은 일반 트랜잭션과 마찬가지로 
액세스하는 테이블에서 Metadata Locks 를 발생시킨다.   

하나의 글로벌 트랜잭션은 그 안에서 트랜잭션 되는 몇가지 action 들이 포함되어 있다.
글로벌 트랜잭션은 하나의 그룹으로 성공적으로 Commit 되거나, 하나의 그룹으로 Rollback 된다.
이러한 동작을 기반으로 ACID 속성을 `up a level` 로 하여 여러 개의 ACID 트랜잭션들이 
하나의 글로벌 트랜잭션의 구성원들로서 조화를 이루어 수행할 수 있게 된다.   

XA Transaction 을 사용하기 위해선 최소 Repeatable Read 수준의 격리 레벨을 요구한다.  
그러나 분산 트랜잭션을 사용하는 경우에는 Seriralizable 수준의 격리 레벨을 사용해야한다.  

1. 하나 이상의 XA Transaction 을 동시에 시작하려고 하면 1400 오류가 발생한다.
2. 일반 트랜잭션이 적용되는 동안 XA 트랜잭션을 시작하려고 할 때 동일한 오류가 발생한다.
3. XA 트랜잭션이 적용되는 동안 일반 트랜잭션을 시작하려고 하면, 1399 오류가 발생한다.
4. XA 트랜잭션이 유효한 경우 일반 트랜잭션에 대해 내제된 COMMIT 를 발생시키는 명령문은 1400 오류






# Galera Cluster에서 이슈
MariaDB Galera Cluster는 XA 트랜잭션을 지원하지 않습니다. 
그러나 MariaDB Galera Cluster 빌드에는 wsrep이라는 내장 플러그인이 포함되어 있습니다. 
MariaDB 10.4.3 이전에는이 플러그인이 내부적으로 XA 가능 스토리지 엔진으로 간주되었습니다. 
MariaDB Galera Cluster에서는 실제 XA를 지원하는 스토리지 엔진이 InnoDB뿐이지만, 
external XA transaction 지원을 동해 XA 트랜잭션이 가능한 경우도 있습니다.

