다양한 엔티티를 생성하여 사용하다 보면 엔티티 생성마다 중복되는 필드(id, createdAt, updatedAt 등)를 반복적으로 계속 생성해 줘야 하는 걸까?라는 의문이 들곤 합니다.   
이번 포스팅에서는 엔티티 생성시 중복되는 필드를 BaseEntity를 활용하여 관리하는 방법에 대해 알아보겠습니다.

# BaseEntity 생성

```java

@Getter
@MappedSuperclass
public class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
}

```

엔티티 생성시 중복되는 속성 id, createdAt, updatedAt을 생성하였습니다.

## @MappedSuperclass란?

@MappedSuperclass는 객체 상속을 통해 사용하지만 상속관계 매핑의 목적보다 공통 매핑 정보가 필요할 때 사용됩니다.   
한마디로 위의 BaseEntity예시와 같이 중복되는 속성을 생성하여 Entity생성 시 상속받아 중복되는 속성을 처리하는 데에 의미를 두고 있습니다.   
여기서 @MappedSuperclass가 해당 BaseEntity를 엔티티로 인식되지 않게 하며, 데이터베이스에 테이블이 생성되지 않게 합니다.

# Entity 생성

```java
// Member Entity

@Entity
@Getter
@Table(name = "members")
public class Member extends BaseEntity {

    @Column(nullable = false, length = 100)
    private String email;

    @Column(nullable = false, length = 60)
    private String password;
}

```

```java
// Todo Entity

@Entity
@Getter
@Table(name = "todos")
public class Todo extends BaseEntity {

    @Column(nullable = false, length = 30)
    private String title;

    private String content;
}

```

중복되는 속성 id, createdAt, updatedAt을 BaseEntity로부터 상속받아 Member와 Todo Entity를 생성하였습니다.

<img width="338" alt="1" src="https://github.com/user-attachments/assets/fcba64a7-aed5-4185-bd0e-822555526f71">

<img width="342" alt="2" src="https://github.com/user-attachments/assets/e6edcc6e-8eb5-4fc7-91ca-de7b9059a7db">

테이블 생성시 BaseEntity로부터 공통 필드 정보가 적용되어 생성되는 것을 확인할 수 있습니다.

Entity생성 시 공통 필드를 BaseEntity로 관리하게 되면 공통 필드를 한 곳에서 정의함으로써 중복코드를 줄일 수 있으며 공통필드의 변경이 필요할 경우 부모 클래스에서만 수정이 이루어져 유지보수 측면에서도
유리할 수 있습니다.   
또한 여러 Entity에서 동일한 필드를 사용할 수 있어 데이터 모델의 일관성을 유지할 수 있습니다.

# Auditing 이란?

Auditing이란 감사, 심사 등의 의미를 가지고 있으며, Spring Data JPA에서는 Auditing기능을 기본적으로 제공합니다.   
이를 사용하여 엔티티가 생성, 수정되는 시점을 감지하여 그 시간과 생성, 수정 한 사람을 기록하하여 그 이력을 남길 수 있습니다.   
본 포스팅에서는 Auditing 기능을 사용하여 생성, 수정되는 시간관리 하는것에 대해 알아보겠습니다.

# Auditing 활성화

먼저 @EnableJpaAuditing 어노테이션을 사용하여 Auditing을 활성화 합니다.   
활성화에는 두가지 방법이 존재합니다.

- Application 클래스에 추가하여 활성화

```java
// 1

@EnableJpaAuditing
@SpringBootApplication
public class TodoListApplication {

    public static void main(String[] args) {
        SpringApplication.run(TodoListApplication.class, args);
    }

}

```

먼저 메인 Application에 @EnableJpaAuditing 어노테이션을 사용하여 Auditing을 활성화 하는 방법입니다.
해당 방법에서는 몇가지 문제점이 존재합니다.

메인 Application에 @EnableJpaAuditing 사용 시 스프링컨테이너가 필요한 테스트를 진행할 때   
<span style="color:red"> JPA metamodel must not be empty! </span>   
해당 에러가 발생할 수 있습니다.   
그 이유는 스프링 의존이 필요한 테스트 진행 시 MainClass가 로딩될 때 @EnableJpaAuditing 어노테이션이 존재하면 JPA 관련 Bean을 불러와야 하는데 mock 테스트 진행 시 해당 Bean이
없어 호출 할 수 없기 때문입니다.   
이 문제는 @MockBean(JpaMetamodelMappingContext.class) 어노테이션을 각 테스트 클래스마다 추가해주는 방법으로도 해결이 가능합니다.

그리고 @DataJpaTest를 이용한 테스트코드 작성 시 Auditing을 활성화 하게 되면 생성, 수정시간 관리를 위해 테스트코드 작성시에도 Auditing 기능을 활성화 시켜주어야 하므로 해당 테스트코드
클래스에 @EnableJpaAuditing 어노테이션이 있는 클래스를 Import 시켜주어야 합니다.

### @DataJpaTest 사용시 ex)

```java

@DataJpaTest
@Import(TodoListApplication.class)
class TodoRepositoryTest {
}

```

이렇게 @EnableJpaAuditing 어노테이션이 존재하는 클래스를 @Import를 통해 추가시켜주어야 하는데 이렇게 메인 Application을 추가하게 되면 스프링의 모든 Bean이 등록 되게 되어
@SpringBootTest와 차이점이 없어져 단일테스트의 의미가 사라지게 됩니다.

앞서 말씀드린 문제점을 해결하기 위한 두번째 Auditing 활성화 방법입니다.

- Configuration 분리

```java
// 2

@EnableJpaAuditing
@Configuration
public class AuditingConfig {
}

```

Configuration 클래스를 생성하여 분리하여 사용합니다.

```java

@DataJpaTest
@Import(AuditingConfig.class)
class TodoRepositoryTest {
}

```

테스트 클래스에 Import 클래스를 다르게 주어 해결이 가능합니다.

# BaseEntity에 추가하기

```java

@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class) // 추가
public class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 기존 어노테이션 @CreationTimestamp
    @CreatedDate // 변경된 어노테이션
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    // 기존 어노테이션 @UpdateTimestamp
    @LastModifiedDate // 변경된 어노테이션
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
}

```

앞서 생성한 BaseEntity와 달라진 점에 대해서 예시 BaseEntity를 작성하였습니다.

## @Entitylisteners

Auditing을 적용할 Entity 클래스에 적용해야하는 어노테이션입니다.   
해당 어노테이션은 Entity의 변화를 감지하여 Entity와 매핑된 테이블의 데이터를 조작합니다.   
파라미터에 이벤트 리스너로 AuditingEntityListener 클래스를 넣어줍니다. 이 클래스는 Spring Data JPA에서 제공하는 이벤트 리스너로 Entity의 영속, 수정 이벤트를 감지하는 역할을
합니다.

## @CreateDate

생성일을 기록하기 위해 LocalDateTime 타입의 필드에 @CreateDate 어노테이션을 사용합니다.   
해당 어노테이션 사용 시 Entity가 생성됨을 감지하고 그 시점을 설정한 필드에 기록합니다.   
생성시간은 수정되지 않도록 해당 컬럼에 updatable = false를 추가하였습니다.

## @LastModifiedDate

수정일을 기록하기 위해 LocalDateTime 타입의 필드에 @LastModifiedDate 어노테이션을 사용합니다.   
해당 어노테이션 사용 시 Entity가 수정됨을 감지하고 그 시점을 설정한 필드에 기록합니다.

<img width="640" alt="3" src="https://github.com/user-attachments/assets/2321ee2b-ce41-42fb-93b0-eccf514a0c15">

회원 생성에 대한 Request 요청 시 생성, 수정시간에 대한 Response가 정상적으로 동작하고 데이터베이스에 동일한 값이 저장되는 모습을 볼 수 있습니다.

# 기존 BastEntity와 Auditing기능을 활용한 BaseEntity와는 무엇이 다를까?

기존에 설정한 BaseEntity와 Auditing기능을 활용한 BaseEntity에는 생성, 수정시간에 사용되는 어노테이션에 차이점이 있습니다.

```java
// 기존
@CreationTimestamp // 생성
@UpdateTimestamp // 수정

// Auditing 기능 활용
@CreatedDate // 생성
@LastModifiedDate // 수정

```

이 두 어노테이션은 몇가지의 주요 차이점이 존재합니다.

1. 소속 라이브러리
    - @CreatedDate, @LastModifiedDate
        - Spring Data JPA의 일부로, JPA Auditing 기능을 통해 자동으로 생성, 수정 날짜를 관리
    - @CreationTimestamp, @UpdateTimestamp
        - Hibernate의 일부로, Hibernate의 Entity 라이프사이클을 기반으로 자동으로 Timestamp를 관리
2. 설정 필요 여부
    - JPA Auditing
        - 활성화 하기 위해 @EnableJpaAuditing 설정 필요
        - AuditorAwre 인터페이스를 구현하여, 생성자나 수정자를 추적하는 기능도 추가 가능
    - Hibernate
        - 별도의 설정 없이 사용 가능
3. 사용 방식
    - JPA Auditing
        - @CreatedDate는 Entity가 처음 저장될 때 해당 필드에 현재 날짜와 시간을 자동으로 관리
        - @LastModifiedDate는 Entity가 업데이트될 때마다 현재 날짜와 시간을 자동으로 갱신
        - 이 두 어노테이션은 필드가 변경될 때 Spring Data JPA가 내부적으로 관리하는 방법으로, 필드가 수동으로 수정되거나 다른 방법으로 저장될 경우 정상적으로 작동되지 않을 수 있습니다.
    - Hibernate
        - @CreationTimestamp는 Entity가 처음 저장될 때만 Timestamp를 설정, 이후 변경되지 않음
        - @UpdateTimestamp는 Entity가 업데이트될 때마다 현재 날짜와 시간으로 자동으로 갱신
        - 이 두 어노테이션은 Hibernate의 엔티티 리스너를 통해 작동하므로, 엔티티의 상태 변화에 즉각 반응
4. 데이터베이스 반영
    - JPA Auditing
        - JPA에서 Entity를 관리하기 때문에, 생성, 수정 시간을 Application 레벨에서 처리하고, 데이터베이스에 반영
        - 높은 추상화를 제공
    - Hibernate
        - Hibernate는 직접적으로 SQL 쿼리를 생성하여 데이터베이스에 반영하므로, 더욱 세밀한 Timestamp 관리 제공
        - Entity의 상태가 변호할 때마다 즉각적으로 반영하여 실시간성 있는 Timestamp 관리가 가능

# 정리

Spring Data JPA의 Auditing은 더욱 많은 기능을 제공하고, Spring의 생태계와 잘 통합되어 있어, Spring을 사용하는 프로젝트에서 많이 활용되며 Hibernate의 Timestamp는
간편하고, 즉각적인 데이터베이스 반영이 필요할 때 유용합니다.

몇가지의 차이점들이 존재하지만 Auditing의 추가적인 기능 (ex: 생성, 수정 사용자 기록 등)을 사용하지않고 오직 시간관리만을 위해 사용한다면 Auditing기능을 활성화 하지 않은 기존의
BaseEntity활용만으로도 간편한 시간 관리가 가능합니다.
