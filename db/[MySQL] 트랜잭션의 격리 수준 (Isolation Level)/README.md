트랜잭션의 격리 수준 (Transaction Isolation Level)은 데이터베이스에서 동시에 실행되는 여러 트랜잭션 간의 상호작용을 제어하여 데이터의 일관성을 유지하고 충돌을 방지하는 데 중요한 역할을 합니다.   
SQL 표준에서 정의된 격리 수준은 트랜잭션 간의 격리 정도와 성능의 균형을 결정하며, 데이터베이스 설계 및 운영에 있어 핵심적인 요소입니다.   
이번 포스팅에서는 트랜잭션의 격리 수준 `Read Uncommitted, Read Committed, Repeatable Read, Serializable`에 대한 특징과 동작 방식에 대해 알아보도록 하겠습니다.

# 트랜잭션의 격리 수준 (Transaction Isolation Level)

트랜잭션의 격리 수준(Transaction Isolation Level)이란 여러 트랜잭션이 동시에 처리될 때, 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 여부를 결정하는 것   
즉, 트랜잭션끼리 얼마나 고립되어 있는지를 나타내는 수준, 한 트랜잭션에서 다른 트랜잭션이 변경한 데이터에 대한 접근 강도를 의미합니다.   

## READ UNCOMMITTED : Level 0

<img width="522" alt="Image" src="https://github.com/user-attachments/assets/81c09290-962f-40ba-8621-95469542c28b" />

READ UNCOMMITTED는 `트랜잭션 내용이 커밋, 롤백과 상관없이 접근`할 수 있는 격리수준 입니다.   
이렇게 트랜잭션의 작업이 완료되기 전 다른 트랜잭션에서 볼 수 있는 현상을 `Dirty Read`라고 합니다. Dirty Read는 데이터가 조회되었다가 사라지는 현상을 일으켜, 시스템에 혼란을 유발할 수 있습니다.   

<img width="557" alt="Image" src="https://github.com/user-attachments/assets/07ed1b93-4fb7-470b-961d-f39331641b5f" />   

롤백 시 트랜잭션이 아직 종료되지 않은 시점에서 다른 트랜잭션이 데이터를 조회가 가능한 것을 볼 수 있는데, 작업중이던 트랜잭션이 롤백되고 난 후 재조회 시 결과가 존재하는 상황이 발생합니다. 이러한 Dirty Read 상황은 시스템에 치명적인 문제를 유발할 수 있습니다.   
그래서 READ UNCOMMITTED는 정합성의 문제가 많은 격리 수준이기 때문에 RDBMS 표준에서는 격리수준으로 인정하지 않습니다.

## READ COMMITTED : Level 1

READ COMMITTED는 트랜잭션의 변경 내용이 `커밋된 데이터만 조회`가 가능합니다. 대부분의 RDBMS에서 기본적으로 사용하는 격리수준입니다.   
한 트랜잭션에서 데이터 변경 시 커밋하기 전 다른 트랜잭션에서 이를 조회 시 실제 테이블 값이 아닌 Undo 영역에 백업된 레코드 값을 가져오게 됩니다. 하지만 하나의 트랜잭션에서 같은 select 쿼리를 실행했을 때 항상 같은 결과를 가져와야하는 REPEATABLE READ의 정합성에 어긋나며, Non-Repeatable Read (반복 읽기 불가능) 문제가 발생합니다.   

<img width="619" alt="Image" src="https://github.com/user-attachments/assets/9c320527-4760-4feb-af71-d2ebd9ce7b31" />

Undo로그는 위 이미지와 같이 한 트랜잭션이 시작되어 데이터 변경 후 아직 커밋하지 않은 상태일 때, 테이블은 먼저 갱신되고 Undo로그로 변경 전의 데이터가 백업되는 것을 의미합니다.   

<img width="646" alt="Image" src="https://github.com/user-attachments/assets/e2823c84-5c5f-4b11-bafd-df53a4214fe8" />   

트랜잭션 커밋 전 다른 트랜잭션에서 이를 조회 시 READ COMMITTED에서는 커밋된 데이터만 조회가 가능하므로, Undo로그에서 변경 전의 데이터를 찾아서 반환하게 됩니다.   
최종적으로 시작된 트랜잭션이 커밋되어야 이후 다른 트랜잭션에서 새롭게 변경된 데이터를 참조할 수 있습니다.   
하지만 READ COMMITTED는 Phantom Read(유령 읽기), Non-Repeatable Read(반복 읽기 불가능) 문제가 발생하게 됩니다.
- `트랜잭션1`에서 커밋하기 전 `트랜잭션2`는 변경된 데이터가 아닌 Undo로그에 있는 데이터를 참조하게 됩니다. `트랜잭션1` 커밋 후에 `트랜잭션2`는 변경된 데이터를 참조하게 되는데 이렇게 반복 읽기 수행 시 다른 트랜잭션의 커밋 여부에 따라 조회 결과가 달라지게 됩니다. 이를 `Non-Repeatable Read`라고 합니다.
- 데이터 삽입을 예로 들었을 때, `트랜잭션1`에서 커밋하기 전 `트랜잭션2`가 조회 시 아무 데이터도 확인할 수 없는데 `트랜잭션1`커밋 후에는 `트랜잭션2`가 조회 시 생성된 데이터를 확인할 수 있습니다. `트랜잭션2` 입장에서는 이전에 없던 데이터가 조회 시 갑자기 나타난 상황인데 위와 같은 상황으로 다른 트랜잭션의 커밋 여부에 따라 데이터 존재 유무가 달라지게 되는데 이를 `Phantom Read`라고 합니다.
- 데이터 수정, 삭제가 이루어질때 `Non-Repeatable Read`, 데이터 삽입 시 `Phantom Read`

## REPEATABLE READ : Level 2

일반 RDBMS에서는 변경 전의 레코드를 Undo공간에 백업합니다. 그러면 변경 전, 후 데이터가 모두 존재하는데 동일한 레코드에 대해 여러 버전의 데이터가 존재하여 이를 `MVCC(Multi-Version Concurrency Control, 다중 버전 동시성 제어)`라고 부릅니다.   
각 트랜잭션은 `순차 증가하는 고유한 트랜잭션 번호`가 존재하며, 백업 레코드에는 어느 트랜잭션에 의해 백업되었는지 `트랜잭션 번호를 함께 저장`합니다. 이후 해당 데이터가 불필요하다고 판단하는 시점에 주기적으로 백그라운드 쓰레드를 통해 삭제합니다.
REPEATABLE READ는 MVCC를 이용하여 한 트랜잭션 내에서 동일한 결과를 보장하지만, 새로운 레코드가 추가되는 경우 `Phantom Read` 부정합이 생길 수 있습니다.

### 동작 흐름
<img width="690" alt="Image" src="https://github.com/user-attachments/assets/c75c270b-485e-461c-a566-0bcb0ed89b30" />

`트랜잭션2`는 `트랜잭션3`이 시작되기 전 먼저 시작된 상태입니다. 이때 REPEATABLE READ는 트랜잭션 번호를 참고하여 자신보다 먼저 실행된 트랜잭션의 데이터만 조회합니다. 테이블에 자신보다 이후에 실행된 트랜잭션의 데이터가 존재할 경우, Undo로그를 참고하여 데이터를 조회합니다.   
`트랜잭션3`이 시작되고 커밋까지 되었지만, `트랜잭션3`은 `트랜잭션2`보다 나중에 실행되었기 때문에 기존과 동일한 데이터를 조회하게 됩니다.   
즉, REPEATABLE READ는 어떤 트랜잭션이 읽은 데이터를 다른 트랜잭션이 수정하더라도 동일한 결과를 보장합니다.

### Phantom Read : 일반 RDBMS
<img width="656" alt="Image" src="https://github.com/user-attachments/assets/d73f34c7-c71f-40b6-b304-3da2c19915f8" />

REPEATABLE READ는 새로운 레코드의 추가를 막지 않아, 조회 시 트랜잭션이 끝나기 전에 다른 트랜잭션에 의해 추가된 레코드가 발견되는데 이를 `Phantom Read(유령 읽기)`라고 합니다.   
하지만 MVCC 기능으로 위 이미지 처럼 일반적인 조회에서 Phantom Read는 발생하지 않습니다. 그 이유는 이미 실행된 트랜잭션보다 나중에 실행된 트랜잭션이 추가한 레코드는 무시하면 되기 때문입니다.

<img width="547" alt="Image" src="https://github.com/user-attachments/assets/9a9d1a93-5c17-44d0-bfff-3547c6368604" />

그러나 일반적인 RDBMS의 경우 위 이미지 처럼 `FOR UPDATE를 통한 쓰기 잠금 (베타적 락)`을 사용 시 Phantom Read가 발생하게 됩니다. `(FOR SHARE 읽기 잠금 (공유 락)에도 발생)`   
잠금 시 MVCC를 통해 Phantom Read가 해결되지 않는 이유는 `잠금이 존재하는 데이터 조회는 Undo로그가 아닌 테이블에서 수행`되기 때문입니다.   
일반적인 조회에서 Undo로그에 잠금을 걸면 되지않나? Undo로그는 이전 데이터를 저장하지만, 데이터 수정 시마다 새로운 로그가 추가(append)되기 때문에 특정 데이터에 잠금을 걸 수 없습니다.`(append-only)`  
따라서 MVCC는 `일반 조회에서 Undo 로그를 참조`해 과거 데이터를 보장하지만, `FOR UPDATE, FOR SHARE는 Undo 로그를 참조하지 않고 테이블에서 데이터를 조회`합니다. 이 때문에 새로운 레코드가 추가된 경우 Phantom Read가 발생할 수 있습니다.

### Phantom Read : MySQL

<img width="480" alt="Image" src="https://github.com/user-attachments/assets/4dceea07-ac6f-4b19-aa8b-54088472de10" />

MySQL의 갭 락(Gap Lock)은 기존 레코드뿐만 아니라 레코드 사이의 `갭(공간)`에도 잠금을 설정하게 되는데, 이는 다른 트랜잭션이 해당 범위 내에서 데이터를 추가하는 것을 방지합니다. 넥스트 키 락(Next-Key Lock)은 레코드 락과 갭 락을 결합한 방식으로 Phantom Read를 방지할 수 있습니다.
즉, 잠금으로 데이터 조회 시 MySQL은 조회한 데이터에는 레코드락, 그보다 큰 범위에는 갭 락으로 넥스트 키 락을 사용하게 됩니다. 따라서 `트랜잭션3`이 INSERT 시도 시 `트랜잭션2의 트랜잭션이 종료될 때 까지 대기 후 트랜잭션이 종료`됩니다. 대기가 너무 길어져 `innodb_ock_wait_timeout` 설정값을 초과하면 타임아웃이 발생해 트랜잭션이 실패하고 롤백됩니다.

<img width="456" alt="Image" src="https://github.com/user-attachments/assets/b786a759-8a08-4ba3-9518-ecbd9a1601d3" />

그렇다면 MySQL Phantom Read는 어떤 상황에 발생할까?   
MySQL의 REPEATABLE READ에서는 일반적으로 Phantom Read가 발생하지 않습니다.   
그러나 위 이미지와 같이 `트랜잭션2`가 일반적인 조회 후 `트랜잭션3`이 INSERT를 통해 데이터 추가 시 잠금이 없으므로 즉시 `트랜잭션 종료(커밋)`됩니다. 이후 `트랜잭션2가 FOR UPDATE를 통해 잠금을 사용하여 조회` 시 Undo로그 대신 테이블의 최신 데이터를 직접 조회하여, `이전에 커밋`된 새로운 레코드를 감지해 Phantom Read가 발생하게 됩니다.   
하지만 거의 사용되지 않는 케이스이며, MySQL이 REPEATABLE READ에서 Phantom Read가 발생하는 거의 유일한 상황으로 볼 수 있습니다.

- SELECT FOR UPDATE 이후 SELECT → 갭 락(Gap Lock)으로 인해 Phantom Read 발생하지 않음
- SELECT FOR UPDATE 이후 SELECT FOR UPDATE → 갭 락(Gap Lock)으로 인해 Phantom Read 발생하지 않음
- SELECT 이후 SELECT → MVCC로 인해 Phantom Read 발생하지 않음
- `SELECT 이후 SELECT FOR UPDATE → Phantom Read 발생`

MySQL REPEATABLE READ는 `갭 락(Gap Lock), 넥스트 키 락(Next-Key Lock)`을 통해 대부분의 Phantom Read를 방지합니다. 그러나 일반적인 조회 후 잠금 조회를 사용하는 특수한 경우에 Phantom Read가 발생하게 됩니다. 그 이유는 `Undo로그 대신 테이블 데이터를 직접 조회하는 MySQL의 동작 특성` 때문입니다.

## SERIALIZABLE : Level 3

SERIALIZABLE는 가장 엄격한 격리 수준이지만, 트랜잭션이 순차적으로 처리되어 동시 처리 성능이 가장 낮습니다.   
MySQL에서 FOR UPDATE, FOR SHARE는 대상 레코드에 각 읽기, 쓰기 잠금을 걸지만, 일반적인 SELECT 작업에서는 레코드 잠금 없이 실행됩니다.   
하지만 SERIALIZABLE 격리 수준에서는 일반적인 SELECT 작업에서도 넥스트 키 락(Next-Key Lock)을 읽기 잠금(Shared Lock, 공유락)으로 설정하여, 한 트랜잭션이 작업을 수행하는 동안 다른 트랜잭션이 추가, 수정 작업을 할 수 없도록 합니다.   
가장 안전하지만 가장 성능이 떨어져 극단적으로 높은 데이터 안정성이 필요한 경우가 아니라면 사용하지 않는 것이 좋습니다.

- 장점: 데이터 일관성을 완벽하게 보장합니다. 모든 트랜잭션이 순차적으로처리되어 교착 상태(Dead Lock)와 데이터 충돌 우려가 없습니다.
- 단점: 동시성(Concurrency)이 매우 낮아, 다중 사용자 환경에서 심각한 성능 저하를 유발할 수 있습니다.

# 마무리

![Image](https://github.com/user-attachments/assets/b8fa711b-ac53-47b0-9445-4dd2febcb1c1)

이번 포스팅에서는 트랜잭션의 격리 수준 `Read Uncommitted, Read Committed, Repeatable Read, Serializable`에 대한 특징과 동작 방식에 대해 알아보았습니다.   
각 격리 수준은 동시성과 데이터 일관성 간의 균형을 어떻게 맞출지에 따라 적합한 상황이 다르며, 각 선택은 성능과 안정성 요구 사항에 따라 달라질 수 있습니다.

| 상호작용                | 설명                                                                                                         |
|---------------------|------------------------------------------------------------------------------------------------------------|
| Dirty Read          | 아직 `커밋되지 않은 다른 트랜잭션의 데이터`를 읽는 것을 의미합니다.                                                                    |
| Non-repeatable Read | `다른 트랜잭션에서 데이터를 수정하고 커밋`했을 때, `동일한 트랜잭션 내에서 이전에 읽은 값과 다르게 조회`되는 것을 의미합니다.<br/>일반적으로 데이터의 수정, 삭제가 이루어질 때 발생 |
| Phantom Read        | `다른 트랜잭션에서 데이터를 삽입하고 커밋`했을 때, `동일한 조건으로 조회 시 이전에 없던 데이터가 새롭게 나타나는 것`을 의미합니다.<br/>일반적으로 데이터의 삽입이 이루어질 때 발생  |

| 레벨 | 격리 수준            | Dirty Read | Non-repeatable Read | Phantom Read |
|----|------------------|------------|---------------------|--------------|
| 0  | READ UNCOMMITTED | 발생         | 발생                  | 발생           |
| 1  | READ COMMITTED   | 없음         | 발생                  | 발생           |
| 2  | REPEATABLE READ  | 없음         | 없음                  | 발생           |
| 3  | SERIALIZABLE     | 없음         | 없음                  | 없음           |

# 참고

https://hoestory.tistory.com/86   
https://mangkyu.tistory.com/299
