---
categories: [Tech, Database]
tags: [Isolation Level, Database, blog]
---


### Transaction Isolation Level 이란

- `Isolation Level` 이란 트랜잭션에서 일관성이 없는 데이터를 허용하는 수준을 이야기한다.
- 동시에 여러 트랜잭션이 처리될 때, 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있도록 허용할지 하지않을지 결정하는 것이다.
- 레벨이 높아질수록 처리에 대한 비용이 증가된다.
  
### 트랜잭션의 격리수준에 따른 문제점

#### 낮은 트랜잭션 격리수준의 문제점

Dirty Read 

- `선행 트랜잭션`에서 A 테이블을 Select 한 후 `후행 트랜잭션`에서 A 테이블을 Update 하는 상황이 발생시, `후행 트랜잭션`이 Commit 하지 않았는데도 `선행 트랜잭션`의 Select에서 변경된 데이터를 읽을 수 있는 상황을 말한다.
- 단순히 말해 commit 되지 않은 변화를 읽는것이다.

Non-Repeatable Read (Fuzzy Read)

- `선행 트랜잭션`이 A 테이블에서 select를 하고 있을때, `후행 트랜잭션`이 A 테이블에 접근하여 테이터를 변경 또는 삭제한 후 commit을 한후, `선행 트랜잭션`이 변경 or 삭제된 데이터를 찾게되는 상황을 말한다.
- 같은 데이터의 값이 하나의 트랜잭션에서 달라짐, 트랜잭션의 속성에 위배됨

Phantom Read

- `선행 트랜잭션`이 A 테이블에서 데이터를 select 한후, `후행 트랜잭션`에서 A 테이블에 데이터를 insert 또는 delete 한 상황에서, `선행 트랜잭션`이 다시 A 테이블의 데이터를 select 할 경우 데이터가 추가되거나 사라지는 현상을 말한다.
- 현상 정리
  - T1(트랜잭션1) 이 A 테이블 select로 10의 데이터를 가진 하나의 튜플을 반환
  - T2에 의해 다른 튜플의 데이터가 10으로 update 되고 commit이 진행됨
  - T1에서 다시 A 테이블을 select시 이전에 없던 데이터를 포함 2개의 튜플이 반환됨  -> Phantom Reard
- 없던 데이터가 갑자기 생기는 현상을 말한다.


> 위의 이상 현상들을 모두 발생하지 않게 처리할 수 있지만 많은 제약사항으로 인해 동시 처리 가능항 트랜젝션의 수가 줄어들어 DB의 전체 처리량 (Throughput)이 하락하게 된다. 이 이상 현상을 허용하는 몇가지 level을 만들어서 사용자가 필요에 따라서 적절하게 선택 할 수 있도록 만들어준것이 `Isolation Level`이라고 한다.


<img data-action="zoom" src='{{"/assets/images/post/isolation-level.png" | relative_url}}' alt='absolute' width="650"/>

level 0 : Read Uncommited

- 트랜잭션이 완료(commit) 되지 않은 상황에서 다른 트랜잭션이 `변경한 데이터를 select 가 가능한 레벨`이다.
- 트랜잭션에서 처리 중이며 커밋되지 않은 데이터의 변경을 다른 트랜잭션이 select 하는 것을 혀용한다. 다시 말해, 완료되지 않은 트랜잭션의 작업 결과를 읽을 수 있는 레벨이다. 그렇기 떄문에 데이터의 일관성을 유지할 수 없다.
- 트랜잭션이 접근하는 데이터에 Lock이 걸리지 않는다.
- Dirty Read, Non-Repeatable Read, Phantom Read 현상이 발생한다.
  
level 1 : Read Committed

- 트랜잭션이 완료(commit) 되지 않은 상황에서 다른 트랜잭션이 `변경한 데이터를 select 가 가능하지 않은 레별`이다.
- 특정 데이터에 대해 트랜잭션이 수행되고 있으면 다른 트랜잭션은 접근할 수 없고 `대기`한다.
- 트랜잭션이 완료시 commit 된 데이터에 한해서 select 가 가능하다.
- Dirty Read 현상이 발생하지 않으므로 `대부분의 DBMS가 기본모드로 채택`하고 있다.
- Non-Repeatable Read, Phantom Read 현상이 발생한다.

level 2 : Repeatable Read

- 트랜잭션 범위 내에서 접근한 데이터의 내용이 항상 `동일함`을 보장해주는 level이다.
- 접근중인 데이터에 대해 다른 트랜잭션에서 변경을 하고 commit을 하였다 하더라도 현재 트랜잭션에서는 그 내용이 반영되지 않는다.
- 하지만 다른 트랜잭션이 새로운 데이터를 삽입했을 때 현재 트랜잭션에서 새로운 데이터에 대한 조회가 가능하다.
- 따라서 한 트랜잭션 안에서 작업을 수행하다가 갑자기 새로운 데이터가 조회될 수 있다. (Phantom Read)

level 3 : Serializable

- 모든 동작이 직렬화 되어 작동한다.
- 트랜잭션이 접근 중인 데이터에 대해 다른 트랜잭션의 모든 접근이 불가능할 뿐만아니라 변경, 삭제 및 새로운 데이터 삽입도 불가능하다.
- 완벽한 일관성을 제공한다. 
