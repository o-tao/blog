웹 서비스에서 페이지네이션은 흔하게 사용되는 기능 중 하나입니다.   
데이터를 효율적으로 조회하기 위해 많은 서비스에서 이 기능을 적용하고 있습니다. 데이터 조회에 대한 기능을 구현하며 페이지네이션 방식 중 offset과 no offset 방식의 페이지네이션에 대해 알게 되었고 이 두 가지 방식의 차이점과 각 방식에 대한 사용성에 대해 정리하고자 합니다.

이번 포스팅에서는 페이지네이션 기능 중 일반적으로 사용되는 `offset`과 `no offset`의 차이점과 각 기능의 장단점, 성능차이에 대해 알아보도록 하겠습니다.

# offset 이란?
페이지네이션의 offset 방식은 주어진 페이지 번호(offset)에 따라 데이터를 조회하는 방법입니다.   
![1](https://github.com/user-attachments/assets/2d7bb7d4-16c6-4c35-8cea-5e092cba17fd)

이 방식은 SQL 쿼리에서 LIMIT와 OFFSET을 사용하여 데이터를 페이지 단위로 나누어 가져옵니다.   
`LIMIT`는 한 페이지에서 가져올 데이터의 수를 정의하고, OFFSET은 특정 위치부터 데이터를 건너뛰고 가져올 지점을 지정합니다.

예를들어 다음과 같은 쿼리가 있을 수 있습니다.
```sql
-- LIMIT → 페이지 사이즈 (해당 페이지에 불러올 데이터의 수)
-- OFFSET → 페이지 번호 (LIMIT의 수에 따라 달라집니다)

-- LIMIT가 1인 경우
SELECT * FROM table ORDER BY id ASC LIMIT 1 OFFSET 0; -- 1페이지
SELECT * FROM table ORDER BY id ASC LIMIT 1 OFFSET 1; -- 2페이지
SELECT * FROM table ORDER BY id ASC LIMIT 1 OFFSET 2; -- 3페이지

-- LIMIT가 10인 경우
SELECT * FROM table ORDER BY id ASC LIMIT 10 OFFSET 0; -- 1페이지
SELECT * FROM table ORDER BY id ASC LIMIT 10 OFFSET 10; -- 2페이지
SELECT * FROM table ORDER BY id ASC LIMIT 10 OFFSET 20; -- 3페이지
```
