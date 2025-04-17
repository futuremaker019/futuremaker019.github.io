---
categories: [Tech, Database]
tags: [Transaction, ACID, Database, blog]
toc: true 
---

### Transaction 이란

Transaction은 단일한 논리적인 작업 단위를 말한다. 논리적인 이유로 여러 SQL문들을 단일 작업으로 묶어서 나눠질 수 없게 만드는 작업이라고 할 수 있다.

하나의 Transaction은 정상적으로 종료된 경우 Commit 연산이 수행되고, 비정상적으로 종료될 경우 Rollback 연산이 수행된다.


#### Commit

Commit 연산은 트랜잭션 처리가 정상적으로 종료되어 트랜잭션이 수행한 변경 내용을 데이터베이스에 반영하는 연산이다.

데이터를 변경한 트랜젝션이 완료되면, 그 트랜잭션에 의해 데이터베이스는 새롭게 일관된 상태로 변경되며, 이 상태는 시스템에 오류가 발생하더라고 취소되지 않는다.

#### Rollback

하나의 트랜잭션 처리가 비정상적으로 종료될시 데이터베이스의 일관성이 꺠졌을 떄 트랜잭션이 행한 모든 변경 작업을 취소하고 이전 상태로 되돌리는 연산이다.

Rollback 연산 시 해당 트랜잭션은 받았던 자원과 잠금(Lock)을 모두 반환하고, 재시작되거나 폐기된다.


### MySQL에서 트랜잭션 처리

```sql
-- mysql 데이터베이스를 사용한다는 것을 명시했다.

-- COMMIT 예제
mysql> START TRANSACTION;
mysql> UPDATE 계좌 set 잔액 = 잔액 - 5000 where 계좌번호 = 100;
mysql> UPDATE 계좌 set 잔액 = 잔액 + 5000 where 계좌번호 = 200;
mysql> COMMIT;  

-- commit은 지금까지 작업한 내용을 DB에 영구적으로 저장한다.
-- transaction을 종료한다.


-- ROLLBACK 예제
mysql> START TRANSACTION;
mysql> UPDATE 계좌 set 잔액 = 잔액 - 5000 where 계좌번호 = 100;
mysql> UPDATE 계좌 set 잔액 = 잔액 + 5000 where 계좌번호 = 200;
mysql> ROLLBACK;  

-- ROLLBACK 명령어 전의 작업들을 모두 취소하고 tranaction 이전의 상태로 돌린다.
-- transaction을 종료한다.
```

### AUTOCOMMIT

각각의 SQL문을 자동으로 tarnsaction 처리 해주는 개념으로 SQL문이 성공적으로 실행하면 자동으로 commit 한다. 

실행중에 문제가 있다면 알아서 ROLLBACK 처리를한다.

MySQL에서는 default로 autocommit이 enabled 되어 있으며 다른 DBMS에서도 대부분 같은 기능을 제공한다.

```sql
mysql> select @@AUTOCOMMIT  -- autocommit의 enabled 여부를 확인할수 있는 명령어

insert into account values ('W', 100000);

-- autocommit이 enabled된 상태이기 떄문에 insert문을 실행하면 자동으로 commit되며 데이터는 영구적으로 저장된다.
```
```sql
mysql> SET autocommit = 0;  -- autocommit을 끔
mysql> DELETE FROM account WHERE balance <= 1000;
mysql> ROLLBACK;            -- 위에서 실행한 delete를 다시 원복시킨다.
```
```sql
mysql> START TRANSACTION;
mysql> UPDATE 계좌 set 잔액 = 잔액 - 5000 where 계좌번호 = 100;
mysql> UPDATE 계좌 set 잔액 = 잔액 + 5000 where 계좌번호 = 200;
mysql> COMMIT; 

-- START TRANSACTION 실행과 동시에 autocommit은 off 된다.
-- COMMIT / ROLLBACK 과 함께 transaction이 종료되면 원래 autocommit 상태로 되돌아간다.
```

### Java에서 Transaction 사용 예제

Transaction의 처리과정을 자바로 코드로 다음과 같이 사용한다.

```java
public void update(String param) {

    // try .. catch 필요
    Context context = new InitialContext();
    DataSource dataSource = (DataSource) context.lookup("java:comp/env/jdbc/mysql");
    Connection connection = dataSource.getConnection();

    try {
        connection.setAutoCommit(false);    // autoCommit을 false하여 transactin start
        
        // ..                               // update 실행 (실행해야할 코드) 

        connection.commit();                // commit 하여 디비에 반영
    } catch (Exception e) {
        // ..

        connection.rollback();              // 장애시 DB rollback

        // ..
    } finally {
        connection.setAutoCommit(true);     // autocommit true로 변경
    }
}
```
 
`@Transactional`을 사용하여 실행코드만 작성하게 만들어준다. 
`@Transactional`은 `AOP`를 사용하여 동작하게 한다. (추후 정리필요)

```java
@Transactional
public void update(String param) {

    // .. 실행코드 작성

}
```
<br>

---

### ACID

Transaction이 갖춰야할 특성은 4가지가 있다. 줄여서 ACID라고 부르며 특징은 아래와 같다.

#### Atomicity (원자성)

- ALL or Nothing
- Transaction은 논리적으로 쪼개질 수 없는 작업 단위이기 떄문에 내부의 SQL문들이 모두 성공해야 한다.
- 중간에 SQL 문이 실패하면 지금까지의 모든 작업을 취소하여 아무일도 없었던 것처엄 ROLLBACK 한다.

작업범위

- DBMS 에서는 commit, rollback을 담당하며 개발자는 언제 commit, rollback을 할지 설정해야 한다.

#### Consistency (일관성)

- Transaction 수행이 성공적으로 완료되면 언제나 일관성 있는 데이터베이스 상태로 변환한다.
- 시스템이 가지고 있는 고정 요소는 트랜잭션 수행 전과 수행 완료 후의 상태가 같아야 한다.
  - ex) A계좌에서 B계좌의 합이 10,000원이면 A계좌에서 B계좌로 5,000월 송금해도 두 계좌의 합은 언제든 같아야 한다.
- Transaction은 DB 상태를 consistent 상태에서 또 다른 consistent 상태로 바꿔줘야 한다.
- constraints, trigger 등을 통해 DB에 정의된 rule을 transaction이 위반했다면 rollback해야 한다.
  - ex) 계좌의 잔고는 0이 될수 없다는 테이블의 constraints를 위반시 rollback 한다.

작업범위

- transaction이 DB에 정의된 rule을 위반했는지 DBMS가 commit 전에 확인하고 알려준다.
- 개발자는 application 관점에서 transaction이 consistent하게 동작하는지 확인해야 한다.

#### Isolation (격리성)

상황: A가 B에게 20만원을 이체할 때 B도 ATM에서 본인 계좌에 30만원을 입급한다면?

<img data-action="zoom" src='{{"/assets/images/post/isolation.png" | relative_url}}' alt='absolute' width="650"/>

> 입금된 30만원은 사라지고 20만원만 추가된 상황

- 여러 transaction들이 동시에 실행될 떄도 혼자 실행되는 것처럼 동작하게 만든다.
- DBMS는 여러 종류의 isolation level 을 제공한다.
- isolation level이 높을수록 동시성 성능이 떨어져 DB의 성능이 줄어들게 된다.
- 개발자는 isolation level 중에 어떤 level로 transaction을 동작시킬지 설정할 수 있다.
- concurrency controi 의 주된 목표가 isolation이다.

#### Durability (지속성)

- 트랜잭션의 실행이 성공적으로 실행 완료된 후에는 시스템에 오류가 발생하더라도 트랜잭션에 의해 변경된 내용은 계속 보존되어야 한다.
- 영구적으로 저장한다. 라고 할 때는 일반적으로 비휘발성 메모리에 저장함을 의미한다.
- 기본적으로 DBMS가 보장한다.

<br>

---

참조

- [https://www.youtube.com/watch?v=sLJ8ypeHGlM](https://www.youtube.com/watch?v=sLJ8ypeHGlM)
- 수제비 (2022) 정보처리기사 필기책


<br>

---

### 추가 공부사항

- `@Transactional` 동작원리
- Isolation level
