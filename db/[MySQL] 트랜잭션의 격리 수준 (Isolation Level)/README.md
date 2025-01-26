트랜잭션의 격리 수준 (Transaction Isolation Level)은 데이터베이스에서 동시에 실행되는 여러 트랜잭션 간의 상호작용을 제어하여 데이터의 일관성을 유지하고 충돌을 방지하는 데 중요한 역할을 합니다.   
SQL 표준에서 정의된 격리 수준은 트랜잭션 간의 격리 정도와 성능의 균형을 결정하며, 데이터베이스 설계 및 운영에 있어 핵심적인 요소입니다.   
이번 포스팅에서는 트랜잭션의 격리 수준 `Read Uncommitted, Read Committed, Repeatable Read, Serializable`에 대한 특징과 동작 방식에 대해 알아보도록 하겠습니다.

# 트랜잭션의 격리 수준 (Transaction Isolation Level)
