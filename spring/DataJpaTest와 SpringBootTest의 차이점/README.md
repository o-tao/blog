# DataJpaTest란?
- JPA관련 구성 요소에 초점을 맞춘 데이터 액세스 계층 테스트
- 특징
  - Application의 JPA에 필요한 Bean을 로드
  - @Repository 어노테이션을 가진 Bean들을 자동 스캔
  - 빠른 테스트 실행 속도
  - 데이터 액세스 레이어에 집중하여 구체적인 테스트 가능
  - in-memory embedded DB 사용 가능

![Alt text](/spring/DataJpaTest와 SpringBootTest의 차이점/images/1.png)   
@DataJpaTest가 포함하고 있는 어노테이션 입니다.

## @TypeExcludeFilters
- Application 컨텍스트가 생성되는 동안 특정 타입의 빈을 제외시키는 역할을 합니다.
- DataJpaTypeExcludeFilter를 통해 JPA외의 @Controller, @Service, @Component 등과 같은 어노테이션이 달린 클래스를 컴포넌트 스캔 대상에서 제외시키는 동작을 수행합니다.

## @AutoConfigureDataJpa
- 테스트 환경에서 Data JPA를 자동으로 구성하는 역할을 합니다.
  - EntityManagerFactory를 생성하여 Entity를 관리
  - 테스트코드에서 Transaction 관리
  - Hibernate와 같은 JPA 구현체에 대한 구성을 수행
  - JPA Repository를 스캔하고 Application 컨텍스트에 등록

## @AutoConfigureTestDatabase
- @DataJpaTest는 기본적으로 내장된 메모리 데이터베이스를 사용하지만 해당 어노테이션을 사용하여 설정을 재정의 할 수 있습니다.
- replace: 어떤 유형의 데이터소스가 테스트 데이터소스로 교체될지 결정 3가지 속성 존재
  - ANY: 기본값으로 어떤 데이터소스가 설정되더라도 테스트 데이터소스로 교체
  - AUTO_CONFIGURED: SpringBoot에 의해 자동 구성된 데이터소스만 대체, 개발자가 구성한 데이터소스는 그대로 두고 SpringBoot가 자동으로 구성한 데이터소스만 테스트 데이터소스로 교체
  - NONE: 어떠한 데이터소스도 테스트 데이터소스로 대체X, 개발자가 구성한 데이터소스 설정을 그대로 사용
- 기본 내장 메모리 데이터베이스 (H2, DERBY, HSQLDB)외에 외부에서 물리적으로 데이터베이스 사용 시 (MySQL 등) 해당 테스트에 아래의 설정이 필요합니다.
```properties
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
```

## @Transactional
@DataJpaTest는 기본적으로 @Transactional상태이며 따로 @Rollback(value=false)를 주지 않으면 테스트가 끝난 후 자동적으로 롤백처리가 됩니다.

---

# SpringBootTest와의 차이점

### @DataJpaTest
- JPA 엔티티, Repository 계층의 테스트
- 데이터베이스 Transaction 및 쿼리 성능 테스트
- 오직 JPA 관련 구성요소만 로드

### @SpringBootTest
- 전체 Application 흐름을 테스트하고자 할 때
- 여러 Bean 간의 상호작용을 검증하고 싶을 때
- 전체 Spring Application 컨텍스트 로드

# 정리
@DataJpaTest, @SpringBootTest의 가장 큰 차이점은 각각의 사용목적에 있습니다.   
데이터 액세스 레이어에 집중하는 경우 @DataJpaTest (단위 테스트)   
Application의 전체적인 통합 테스트를 하는 경우 @SpringBootTest (통합 테스트)

> @DataJpaTest를 이용한 테스트코드 작성 예시
[Tistory](https://tao-tech.tistory.com/9)
[GitHub](https://github.com/o-tao/blog/tree/main/springboot/%40DataJpaTest%EB%A5%BC%20%EC%9D%B4%EC%9A%A9%ED%95%9C%20%ED%85%8C%EC%8A%A4%ED%8A%B8%EC%BD%94%EB%93%9C%20%EC%9E%91%EC%84%B1%20deleteAll%20deleteAllInBatch)
