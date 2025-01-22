많은 애플리케이션과 서비스는 데이터를 효과적으로 저장하고 관리하기 위해 데이터베이스를 사용합니다. 사용자가 동시에 데이터를 처리하는 상황에서 데이터의 무결성과 일관성을 유지 문제를 해결하기 위해 트랜잭션이 사용됩니다.   
이번 포스팅에서는 트랜잭션의 개념과 특징, 동작원리에 대해 알아보도록 하겠습니다.

# 트랜잭션(Transaction)

<img width="530" alt="Image" src="https://github.com/user-attachments/assets/fafc895f-d16b-4659-a845-1e139a4a1bad" />   

트랜잭션(Transaction)은 데이터베이스 관리 시스템(DBMS)에서 데이터를 처리하기 위해 사용되는 작업의 단위입니다.   
데이터 처리로는 `INSERT, SELECT, UPDATE, DELETE`와 같은 쿼리를 통한 연산 수행을 의미합니다. 사전적 의미로는 `거래`를 뜻하며, 데이터베이스에서 트랜잭션은 하나의 거래를 안전하고 일관성있게 처리하도록 보장해주는 것이라고 할 수 있습니다.    
데이터베이스에서 트랜잭션은 하나의 논리적인 작업 단위로 간주되며, 여러 작업(쿼리)이 하나의 트랜잭션 내에서 수행됩니다. 이러한 트랜잭션은 모두 성공적으로 완료되어야만 적용되며, 하나라도 실패하면 전체 작업이 `취소(Rollback)`되어 데이터의 일관성을 유지합니다.

트랜잭션 유무를 통해 더욱 자세히 알아보겠습니다.

## ex : 트랜잭션 미적용

<img width="877" alt="Image" src="https://github.com/user-attachments/assets/32b03ed0-afab-47a0-bc6d-45e38e45670c" />

트랜잭션이 적용되지 않은 경우, `USER1`의 계좌에서 금액이 차감된 후 `USER2`의 계좌에 금액이 이체되지 않은 상태에서 오류가 발생하면, 전체 거래가 일관성을 잃게 됩니다.   
즉, `USER1`의 계좌는 이미 금액이 감소했지만, `USER2`의 계좌에는 증가된 금액이 반영되지 않은 상태로 저장되어 치명적인 문제로 다가오게 됩니다. 

이러한 문제를 방지하기 위해 트랜잭션을 적용할 수 있습니다.

## ex : 트랜잭션 적용

<img width="662" alt="Image" src="https://github.com/user-attachments/assets/0420e8c5-52da-47bc-b887-eb0db9756648" />

DB가 제공하는 트랜잭션 기능을 사용하면 `커밋(commit)과 롤백(rollback)`으로 정상적인 작업을 수행할 수 있습니다.   
작업이 문제없이 수행되면 `commit`을 호출하여 정상적인 저장이 이루어지고, 작업 실행 중 문제가 발생하였을 경우 기존의 상태로 돌아갈 수 있는데, 작업 중 하나라도 실패해서 이전으로 되돌리는 작업이 `rollback`입니다.   
롤백 후 결과적으로 `USER1`의 계좌 잔고는 감소하지 않아 전체 거래에서 일관성을 유지할 수 있습니다.

### 트랜잭션 동작 과정

<img width="871" alt="Image" src="https://github.com/user-attachments/assets/d5688aa7-5ac4-4d6f-987c-b68e8997d567" />   

1. 트랜잭션 시작
   - `USER1`의 계좌에서 금액을 차감하고, `USER2`의 계좌에 금액을 추가하는 두 작업이 하나의 트랜잭션으로 시작됩니다. 트랜잭션이 시작되면, 이 두 작업은 묶여서 하나의 단위로 처리됩니다.
2. 작업 실행
   - `USER1` 계좌에서 금액을 차감하는 작업 완료 → `USER2` 계좌에 금액을 추가하는 작업 진행
3. 오류 발생 시 롤백
   - 만약 `USER2`의 계좌에 금액을 추가하는 도중 오류가 발생하면, 트랜잭션은 롤백됩니다. 롤백을 호출하면 `USER1`계좌에서 차감된 금액도 되돌려지고, 두 계좌의 상태는 트랜잭션 시작 이전으로 되돌려집니다. 이는 데이터의 일관성을 유지합니다.   
4. 성공 시 커밋
   - 만약 두 작업이 모두 성공적으로 처리되면, 트랜잭션을 커밋하여 변경 사항을 데이터베이스에 영구적으로 반영합니다. 이때 `USER1`의 계좌는 정상적으로 차감되고, `USER2`의 계좌에는 금액이 추가된 상태로 확정됩니다.

# 트랜잭션의 특징 (ACID)

트랜잭션은 데이터베이스의 일관성과 무결성을 보장하기 위해 `ACID`속성을 따라야합니다. 각 속성을 통해 데이터베이스 작업이 신뢰성과 안정성을 유지할 수 있습니다.

- 원자성 (Atomicity)
  - 트랜잭션 내의 모든 작업은 하나의 단위로 처리됩니다. 즉, 한 단위의 모든 작업이 성공적으로 완료 되는 경우 `커밋`되고, 하나라도 실패하는 경우 `롤백`됩니다. 이를 통해 작업이 모두 실행되거나 아무것도 실행되지 않는 상태를 보장합니다.
  - ex: 계좌 이체 시 송금 계좌에서 금액이 출금되고 수취 계좌에 금액이 입금되는데, 두 작업 중 하나라도 실패하면 전체 이체가 취소됩니다.

- 일관성 (Consistency)
  - 트랜잭션이 완료되면 데이터베이스는 일관성 있는 상태를 유지해야 합니다. 데이터베이스의 규칙을 따르며, 트랜잭션이 끝날 때까지 데이터는 유효해야 합니다.
  - ex: 트랜잭션이 진행되는 동안에 데이터베이스가 변경되어도 `업데이트된 데이터베이스로 트랜잭션이 진행되는 것이 아닌`, `처음 트랜잭션을 ㅈ린행하기 위해 참조한 데이터베이스로 진행`됩니다.

- 격리성 (Isolation)
  - 트랜잭션은 독립적으로 수행되어야 하며, 다른 트랜잭션의 영향을 받지 않아야 합니다. 여러 트랜잭션이 동시에 실행되는 경우에도, 각 트랜잭션은 서로 격리되어 진행됩니다.
  - ex: 한 사용자가 상품 재고를 변경하는 동안, 다른 사용자는 동일 상품의 재고를 확인하거나 변경할 수 없습니다. 트랜잭션이 끝난 후 결과를 확인할 수 있습니다.

- 지속성 (Durability)
  - 트랜잭션이 성공적으로 완료되면, 그 결과는 시스템 장애나 재시작 후에도 지속적으로 보존됩니다. 데이터베이스에 반영된 내용은 영구적으로 저장됩니다.
  - ex: 트랜잭션이 커밋된 후 서버가 갑작스럽게 종료되더라도 데이터는 손실되지 않고 복구됩니다.

이 네 가지 속성을 통해 데이터베이스의 `일관성, 무결성, 안정성`을 유지할 수 있습니다.

# 코드 예시 : 커밋

```sql
START TRANSACTION; -- 트랜잭션 시작

SELECT * FROM accounts; -- 초기 상태
UPDATE accounts SET money = money - 2000 WHERE id = 'USER1'; -- 금액 차감
SELECT * FROM accounts; -- 금액 차감 후 상태
UPDATE accounts SET money = money + 2000 WHERE id = 'USER2'; -- 금액 추가
SELECT * FROM accounts; -- 금액 추가 후 상태

COMMIT; -- 커밋

SELECT * FROM accounts; -- 결과
```

<img width="288" alt="Image" src="https://github.com/user-attachments/assets/889e0b4e-e762-472d-9f0e-9ce89b9285d7" />
<img width="289" alt="Image" src="https://github.com/user-attachments/assets/45e95b2e-40d4-4b8c-882e-2626564c14c4" />
<img width="288" alt="Image" src="https://github.com/user-attachments/assets/e93b17b6-9dec-4251-b5e6-33f2636ccafd" />
<img width="287" alt="Image" src="https://github.com/user-attachments/assets/0ab3b50e-f20d-4602-932c-9ae5a2f03445" />

해당 작업의 결과는 다음과 같습니다. 각 명령을 수행 후 `commit` 호출 시 데이터베이스에 영구적으로 저장됩니다. `commit`을 호출하기 전까지 데이터베이스에 영구적으로 반영되지 않습니다.

# 코드 예시 : 롤백

```sql
START TRANSACTION; -- 트랜잭션 시작

SELECT * FROM accounts; -- 초기 상태
UPDATE accounts SET money = money - 2000 WHERE id = 'USER1'; -- 금액 차감
SELECT * FROM accounts; -- 금액 차감 후 상태
UPDATE accounts SET money = money + 2000 WHERE id = 'USER2'; -- 금액 추가
SELECT * FROM accounts; -- 금액 추가 후 상태

ROLLBACK; -- 트랜잭션을 취소하고 START TRANSACTION 실행 전 상태로 롤백

SELECT * FROM accounts; -- 결과
```

<img width="288" alt="Image" src="https://github.com/user-attachments/assets/889e0b4e-e762-472d-9f0e-9ce89b9285d7" />
<img width="289" alt="Image" src="https://github.com/user-attachments/assets/45e95b2e-40d4-4b8c-882e-2626564c14c4" />
<img width="288" alt="Image" src="https://github.com/user-attachments/assets/e93b17b6-9dec-4251-b5e6-33f2636ccafd" />
<img width="288" alt="Image" src="https://github.com/user-attachments/assets/889e0b4e-e762-472d-9f0e-9ce89b9285d7" />

트랜잭션 시작 후 한나의 작업단위로 수행되며 마지막 까지 상태를 확인했을 때 테이블에 데이터가 반영되어 있었지만, `롤백` 후 트랜잭션이 시작하기 전 초기상태로 되돌아간 것을 확인할 수 있습니다.

# 트랜잭션 예외

DDL `CREATE, DROP, ALTER, RENAME, TRUNCATE`는 트랜잭션의 롤백 대상이 아니며, 모든 작업에 대해 트랜잭션의 롤백 명령이 적용되는 것은 아닙니다.
- 자동 커밋
  - 대부분의 데이터베이스에서는 DDL 명령 `CREATE, DROP, ALTER, RENAME, TRUNCATE` 이 실행될 때 자동으로 커밋됩니다. 따라서 DDL 명령이 실행된 후에는 이전 트랜잭션 상태로 롤백할 수 없습니다.
- 트랜잭션 분리
  - DDL 명령은 트랜잭션과 별개로 처리되며, 실행 즉시 데이터베이스에 반영됩니다.
- 일부 데이터베이스(MySQL, Oracle 등)에서는 DDL 명령을 트랜잭션 내부에 포함하려고 하면 트랜잭션이 자동으로 종료되거나 오류가 발생할 수 있습니다.

autocommit이 활성화 되어 있는 경우
- MySQL에서 기본적으로 `autocommit` 옵션이 켜져 있다면, 각 SQL 명령이 실행 후 `자동 커밋`되며, 트랜잭션이 명시적으로 시작되지 않는다면 롤백이 불가능합니다.
```sql
SET autocommit = 0; -- 오토커밋 false 설정
START TRANSACTION -- 트랜잭션 명시적 호출
```
오토커밋을 설정에서 해제하거나, 트랜잭션을 명시적으로 호출하여 해결이 가능합니다.

# 마무리

이번 포스팅에서는 트랜잭션의 개념과 특징, 동작원리에 대해 알아보았습니다.    
트랜잭션은 데이터베이스의 신뢰성과 일관성을 보장하는 핵심 개념입니다. `ACID` 특성을 통해 복잡한 데이터 작업에서도 무결성을 유지하며, 오류와 충돌 상황에서도 안전한 처리를 보장할 수 있습니다.
