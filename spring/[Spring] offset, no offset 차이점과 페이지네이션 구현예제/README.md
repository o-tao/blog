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

# SpringDataJPA offset Pagination 예제코드

Spring Data JPA를 사용해 offset 페이지네이션을 구현한 예시입니다.

예제코드에서는 TodoList의 제목(title)을 조회하는 것으로 진행하며, SpringDataJPA로만 구현 한 예제와 QueryDSL을 사용한 구현 예제를 다루도록 하겠습니다.

SpringDataJPA는 자동으로 페이징 처리된 결과를 제공해 주는 편리한 기능을 제공하며, `PageRequest`를 통해 간단히 페이징 요청을 할 수 있습니다.   
이 방식은 SpringDataJPA의 기본 기능을 사용하기 때문에 구현이 간단하며, 추가적인 설정 없이도 페이지네이션을 구현할 수 있는 장점이 있습니다.

offset 예제코드에서 응답되는 JSON은 다음과 같습니다.

```json
// Response 예시

{
    "contents": [
        {
            "title": "todo title1",
            "status": "TODO",
            "createdAt": "2024-10-18 04:25:17",
            "updatedAt": "2024-10-18 04:25:17"
        },
        ... // 나머지 리스트
    ],
    "page": 1, // 현재 페이지 (확인용)
    "size": 10, // 페이지에 불러올 데이터 수 (확인용)
    "totalPages": 3, // 총 페이지 수
    "totalElements": 30 // 총 데이터 수
}
```

## Entity
```java
@Entity
@Getter
@Table(name = "todos")
public class Todo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 30)
    private String title;

    @Column(nullable = false, length = 250)
    private String content;

    @Enumerated(EnumType.STRING)
    private TodoStatus status;
    
    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
    
    }
```

```java
// todo status에 대한 enum

@Getter
@RequiredArgsConstructor
public enum TodoStatus {
    TODO("할일"),
    DONE("완료");

    private final String type;
}
```

## Repository

```java
public interface TodoRepository extends JpaRepository<Todo, Long> {

}
```

## Request DTO

```java
@Getter
@Setter
public class OffsetRequest {

    @Positive(message = "페이지는 양수여야 합니다.")
    private int page = 1;

    @Positive(message = "한 페이지에 조회 할 데이터 수는 양수여야 합니다.")
    private int size = 5;
}
```
요청받을 page, size에 대한 기본값을 상수로 넣어 Request DTO를 생성하였습니다.   
상수로 값을 지정하더라도 요청에 따라 값 지정이 가능합니다.

```bash
// 요청 예시
localhost:8080/offset // 상수로 지정한 기본값 적용

localhost:8080/offset?page=10&size=20 // page와 size 값을 함께 요청하여 지정
```

## Response DTO

```java
@Getter
@Builder
public class OffsetResponse {

    String title;
    TodoStatus status;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    LocalDateTime createdAt;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    LocalDateTime updatedAt;

    public static OffsetResponse of(Todo todo) {
        return OffsetResponse.builder()
                .title(todo.getTitle())
                .status(TodoStatus.TODO)
                .createdAt(todo.getCreatedAt())
                .updatedAt(todo.getUpdatedAt())
                .build();
    }
}
```
조회 시 응답될 값을 Response DTO로 생성하였습니다.

```java
@Getter
public class OffsetInfoResponse<T> {

    private final List<T> contents;
    private final int page;
    private final int size;
    private final long totalPages;
    private final long totalElements;

    private OffsetInfoResponse(List<T> contents, int page, int size, long totalPages, long totalElements) {
        this.contents = contents;
        this.page = page;
        this.size = size;
        this.totalPages = totalPages;
        this.totalElements = totalElements;
    }

    public static <T> OffsetInfoResponse<T> of(List<T> contents, int page, int size, long totalPages, long totalElements) {
        return new OffsetInfoReesponse<>(contents, page, size, totalPages, totalElements);
    }
}
```
조회 요청 시, 응답 값을 List형태로 담아 하나의 객체(contents)로 반환하는 제네릭 클래스를 생성하였습니다.   
위에서 지정한 OffsetResponse의 값을 contents에 담아 해당 OffsetInfoResponse의 값이 결과적으로 응답됩니다.

## Service

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OffsetService {

	private final TodoRepository todoRepository;

	public OffsetInfoResponse<OffsetResponse> offsetPagination(int page, int size) {
        Page<Todo> all = todoRepository.findAll(PageRequest.of(page - 1, size));
        return OffsetInfoResponse.of(
                all.stream().map(OffsetResponse::of).toList(),
                page,
                size,
                all.getTotalPages(),
                all.getTotalElements()
        );
    }
}
```
`todoRepository.findAll(PageRequest.of(page - 1, size))`를 통해 전체 데이터를 페이징처리하고 있습니다.   
PageRequest는 page와 size를 받아 요청 페이지와 데이터 개수를 지정합니다.

Service 클래스에서 제네릭 클래스를 사용하여 리스트 형태의 데이터를 반환하고 있습니다. 요청받은 페이지 번호와 사이즈, 총 페이지 수, 그리고 총 데이터 수를 함께 반환합니다.

## Controller

```java
@RestController
@RequiredArgsConstructor
public class Controller {

    private final OffsetService offsetService;

    @GetMapping("/offset")
    public OffsetInfoResponse<OffsetResponse> offsetPagination(@Valid OffsetRequest offsetRequest) {
        return offsetService.offsetPagination(offsetRequest.getPage(), offsetRequest.getSize());
    }
}
```
Controller 클래스는 클라이언트에서 받은 요청을 페이지 번호와 사이즈 값을 객체로 받아 Service 클래스에 전달합니다.   
요청에 따라 OffsetInfoReesponse<OffsetResponse> 형식의 페이징 정보를 응답합니다.

<img width="498" alt="2" src="https://github.com/user-attachments/assets/61553bcb-203e-4432-9d01-16ff416063d3" />

page: 현재 페이지   
size: 현재 페이지에서 보여줄 데이터의 개수   
totalPages: 총 페이지 생성 개수   
totalElements: 총데이터의 개수

기본 url 호출 시 초기에 세팅한 page와 size를 적용하며 contents에 담긴 데이터 리스트가 표시되고 그에 따른 페이지가 생성되는 모습을 볼 수 있습니다.

<img width="455" alt="3" src="https://github.com/user-attachments/assets/88526340-2a62-41a6-99a1-307963960de2" />

기본세팅한 값이 아닌 값을 요청하여 호출할 경우 그에 따른 페이지와 size에 따른 페이지수를 적용하는 모습을 볼 수 있습니다.   

# JPA + QueryDSL offset Pagination 예제코드

QueryDSL을 사용한 offset 예제코드에서도 응답되는 JSON 동일하며,   
로직이 변경되지 않는 Entity, Request DTO, Controller를 제외하고 변경되는 로직을 중점적으로 작성하도록 하겠습니다.

## Config

```java
@Configuration
public class QueryDslConfig {

    @PersistenceContext
    private EntityManager entityManager;

    @Bean
    public JPAQueryFactory queryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

먼저 QueryDSL을 사용하기 위한 Config 클래스를 생성합니다.

`@Configuration` Spring이 이 클래스를 설정 클래스(Bean을 정의하는 클래스)로 인식하기 위한 어노테이션   
`@PersistenceContext` JPA의 EntityManager를 주입하기 위한 어노테이션   
`JPAQueryFactory` QueryDSL에서 제공하는 쿼리 생성기, JPA와 함께 사용되어 안전한 쿼리를 작성하게 해 줍니다.   
`@Bean` 어노테이션을 통해 Spring 컨텍스트에 등록되며, 다른 클래스에서 주입받아 사용이 가능합니다.

## QueryRepository

```java
@Repository
@RequiredArgsConstructor
public class QueryRepository {

    private final JPAQueryFactory jpaQueryFactory;

    public List<Todo> findByTodos(int page, int size) {
        return jpaQueryFactory
                .selectFrom(todo)
                .offset((long) (page - 1) * size)
                .limit(size)
                .fetch();
    }

    public Long countByTodo() {
        return jpaQueryFactory
                .select(todo.count())
                .from(todo)
                .fetchOne();
    }
}
```

- findByTodos 메서드
해당 메서드는 페이지네이션을 통해 Todo 엔티티 리스트를 반환합니다.   
`jpaQueryFactory.selectFrom(todo)`는 `Todo` 엔티티를 선택하는 기본 쿼리의 시작점입니다.   
`offset((long) (page - 1) * size)`는 SQL의 OFFSET을 설정합니다. 페이지 번호에 따라 어느 데이터부터 가져올지를 결정합니다.    
-1을 넣어줌으로써 0페이지가 아닌 1페이지부터 시작하게 작성하였습니다.   
`limit(size)`는 SQL의 LIMIT을 설정하여 한 번에 가져올 데이터의 수를 제한합니다.   
`fetch()`는 실행된 쿼리의 결과를 List로 반환합니다.

findByTodos 메서드의 SQL 쿼리는 다음과 같습니다.
```sql
-- page가 1이고 size가 10 일 경우
SELECT * FROM todos ORDER BY id LIMIT 10 OFFSET 0;  -- 1페이지
```

- countByTodo 메서드
해당 메서드는 Todo 엔티티의 총개수를 반환합니다.   
`select(todo.count())`는 Todo의 개수를 선택하는 쿼리입니다.   
`fetchOne()`은 결과를 단일 값으로 반환합니다. 여기서는 Todo의 총개수를 반환하게 됩니다. (totalElements)

countByTodo 메서드의 SQL 쿼리는 다음과 같습니다.
```sql
SELECT COUNT(*) FROM todos;
```

## Response DTO

```java
@Getter
public class OffsetInfoResponse<T> {

    private final List<T> contents;
    private final int page;
    private final int size;
    private final long totalPages;
    private final long totalElements;

    private OffsetInfoResponse(List<T> contents, int page, int size, long totalElements) {
        this.contents = contents;
        this.page = page;
        this.size = size;
        this.totalPages = (int) Math.ceil((double) totalElements / size); // 총 페이지 계산
        this.totalElements = totalElements;
    }

    public static <T> OffsetInfoResponse<T> of(List<T> contents, int page, int size, long totalElements) {
        return new OffsetInfoReesponse<>(contents, page, size, totalElements);
    }
}
```
Response DTO는 이전에 SpringDataJPA로만 구현한 예제코드와 동일하며 변경점은   
PageRequest.of를 사용하지 않고 값을 전달하여 totalPages를 매개변수로 받지 않고 클래스 내에서 직접 계산하여 적용하였습니다.

## Service
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OffsetService {

    private final QueryRepository queryRepository;

    public OffsetInfoResponse<OffsetResponse> offsetPagination(int page, int size) {
        List<Todo> todos = queryRepository.findByTodos(page, size);
        long totalElements = queryRepository.countByTodo();
        return OffsetInfoResponse.of(
                todos.stream().map(OffsetResponse::of).toList(),
                page,
                size,
                totalElements
        );
    }
}
```
Service 클래스에서는 QueryRepository에서 작성한 메서드로 엔티티 리스트, 총 데이터 수를 생성하여 응답합니다.   
SpringDataJPA 구현예제와의 변경점은 PageRequest.of를 사용하지 않고 직접 값을 매개변수로 받아 사용하고 있으며 QueryDSL을 통해 직접적으로 Todo 엔티티 리스트, 총 데이터 수를 생성하여 반환하고 있습니다.

QueryDSL을 통한 예제코드 역시 SpringDataJPA의 예제코드와 동일한 결과를 보여줍니다.

예제코드를 통해 offset 구현방식에 대해 알아보았는데 중간에 페이지 값을 계산한 식이 있었는데 페이지네이션 값 계산을 정리하자면 다음과 같습니다.

## Pagination 값 계산

- 총 페이지 개수 = Math.ceil(전체 컨텐츠 개수 / 한 페이지에 보여줄 컨텐츠의 개수)
- 화면에 보여질 페이지 그룹 = Math.ceil(현재 페이지 번호 / 한 화면에 보여줄 페이지의 개수)
- 화면에 보여질 페이지의 첫 번째 페이지 번호 = ((페이지 그룹 번호 - 1) * 한 화면에 보여줄 페이지의 개수) + 1
- 화면에 보여질 페이지의 마지막 페이지 번호 = 페이지 그룹 번호 * 한 화면에 보여줄 페이지의 개수 단, 페이지 그룹 번호 * 한 화면에 보여줄 페이지의 개수가 전체 페이지 개수보다 크다면 전체 페이지가 된다

## offset 정리

offset은 주어진 페이지 번호에 따라 데이터를 조회하는 방법으로, LIMIT와 OFFSET을 사용하여 데이터를 페이지 단위로 나누어 가져오는 방식입니다.   
이는 사용자가 페이지를 지정하여 원하는 페이지를 즉시 조회할 수 있는 장점이 있습니다.   
하지만 페이지 조회 시 데이터를 처음부터 읽어와 원하는 페이지를 보여주게 되는데 이는 다량의 데이터처리 시 데이터가 많아질수록 처리속도가 점점 저하될 수 있다는 단점이 있습니다.

# no offset 이란?

No offset 페이지네이션은 대량의 데이터를 처리할 때 성능을 최적화하기 위한 페이징 방식입니다.

![4](https://github.com/user-attachments/assets/bc118fad-549c-4e32-8231-4246ba5d6c82)

이 방식은 이전 페이지의 마지막 데이터를 기준으로 다음 데이터를 가져옵니다. OFFSET을 사용하지 않으며, WHERE 절을 이용해 특정 기준 (마지막 데이터의 ID나 타임스탬프 등)을 설정한 후 데이터를 조회합니다.

예를 들어 다음과 같은 쿼리가 있을 수 있습니다.

```sql
SELECT * FROM table ORDER BY id DESC LIMIT 10; -- 1페이지 조회 (첫 페이지)
```

이 쿼리는 가장 최근에 삽입된 데이터 10개를 반환합니다. 여기서 첫 번째 페이지에 해당하는 데이터가 선택됩니다.

실제 데이터베이스 테이블 조회 시 결과는 다음과 같습니다.

<img width="640" alt="5" src="https://github.com/user-attachments/assets/03c8781c-8700-44ae-9a22-9fe7210fd7a8" />

이 중에서 마지막으로 조회된 id는 491입니다.

```sql
SELECT * FROM table WHERE id < 491 ORDER BY id DESC LIMIT 10; -- 2페이지 조회
```

다음 페이지 조회 시 이전페이지에서 마지막으로 조회된 데이터 (id)를 기준으로, 그보다 작은 데이터를 가져옵니다.   
이때 마지막 id (또는 다른 기준 값)을 WHERE 조건에 추가하여 다음 데이터를 조회합니다.

조회 결과는 다음과 같습니다.   

<img width="640" alt="6" src="https://github.com/user-attachments/assets/68fdee59-34e9-4169-befa-0f8143acd349" />

페이지 별 쿼리를 정리하자면 다음과 같습니다.

```sql
SELECT * FROM table ORDER BY id DESC LIMIT 10; -- 첫 페이지
SELECT * FROM table WHERE id < :lastId ORDER BY id DESC LIMIT 10; -- 다음 페이지
```

다음으로 예제코드를 통하여 알아보도록 하겠습니다.

# JPQL no offset Pagination 예제코드

위의 예제코드와 마찬가지로 TodoList의 제목(title)을 조회하는 것으로 진행하며, JPQL로 구현 한 예제와 QueryDSL을 사용한 구현 예제를 다루도록 하겠습니다.   

no offset 예제코드에서 응답되는 JSON은 다음과 같습니다.

```json
// Response 예시
 
{
    "contents": [
        {
            "id": 1,
            "title": "todo title",
            "status": "TODO",
            "createdAt": "2024-10-18 04:25:17",
            "updatedAt": "2024-10-18 04:25:17"
        },
        ... // 나머지 리스트
    ],
    "size": 5, // 한 페이지당 보여줄 데이터의 수 (확인용)
    "hasData": true, // 현재 페이지의 데이터 존재여부
    "hasNext": true // 다음 페이지의 데이터 존재여부
}
```

## Entity

```java
@Entity
@Getter
@Table(name = "todos")
public class Todo {
 
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
 
    @Column(nullable = false, length = 30)
    private String title;
 
    @Column(nullable = false, length = 250)
    private String content;
 
    @Enumerated(EnumType.STRING)
    private TodoStatus status;
    
    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
 
    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
    
    }
```

Entity는 전과 동일합니다.

## Repository

```java
public interface TodoRepository extends JpaRepository<Todo, Long> {
 
    // 첫 페이지: 최근 항목 n개 가져오기
    @Query("SELECT t FROM Todo t ORDER BY t.id DESC")
    List<Todo> findFirstPage();
 
    // 다음 페이지: 마지막 id보다 작은 항목을 n개 가져오기
    @Query("SELECT t FROM Todo t WHERE t.id < :lastId ORDER BY t.id DESC")
    List<Todo> findNextPage(@Param("lastId") Long lastId);
}
```

첫 페이지 호출 메서드와 다음 페이지 호출 메서드를 분리하여 다루었습니다.   
첫 페이지 호출 시 최근 항목 데이터를 지정한 size만큼 가져옵니다.   
다음 페이지 호출 시 마지막 id (또는 다른 기준 값) 보다 작은 항목을 지정한 size만큼 가져옵니다.   
default size = 5

JPQL에서는 LIMIT에 대한 조건을 지원하지 않으므로 필자는 자바코드 내에서 List에 LIMIT를 주었습니다.   
Pageable을 사용하는 방법도 있지만 no offset방식의 구현에서는 Pageable을 사용하여 페이지에 대한 설정을 직접 할 수 있다는 것에 대한 가능성을 열어두기보다 Request값을 지정하여 받는 방식으로 구현하였습니다.

## Request DTO

```java
@Getter
@Setter
public class NoOffsetRequest {
 
    @Positive(message = "데이터의 id는 양수여야 합니다.")
    private Long lastId;
 
    @Positive(message = "한 페이지에 조회 할 데이터 수는 양수여야 합니다.")
    private int size = 5;
}
```

LastId와 size를 요청받도록 추가하였고 Validation 처리하였습니다.

## Response DTO

```java
@Getter
@Builder
public class NoOffsetResponse {
 
    Long id;
    String title;
    TodoStatus status;
 
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    LocalDateTime createdAt;
 
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    LocalDateTime updatedAt;
 
    public static NoOffsetResponse of(Todo todo) {
        return NoOffsetResponse.builder()
                .id(todo.getId())
                .title(todo.getTitle())
                .status(todo.getStatus())
                .createdAt(todo.getCreatedAt())
                .updatedAt(todo.getUpdatedAt())
                .build();
    }
}
```

offset때와 동일하지만 no offset 방식으로 조회할 때 좀 더 가독성이 좋게 하기 위해 id값을 추가하였습니다.

```java
@Getter
public class NoOffsetInfoResponse<T> {
 
    private final List<T> contents;
    private final int size;
    private final boolean hasData;
    private final boolean hasNext;
 
    public NoOffsetInfoResponse(List<T> contents, int size, boolean hasData, boolean hasNext) {
        this.contents = contents;
        this.size = size;
        this.hasData = hasData;
        this.hasNext = hasNext;
    }
 
    public static <T> NoOffsetInfoResponse<T> of(List<T> contents, int size, boolean hasData, boolean hasNext) {
        return new NoOffsetInfoResponse<>(contents, size, hasData, hasNext);
    }
}
```

offset 예제코드와 마찬가지로 조회 요청 시, 응답 값을 List형태로 담아 하나의 객체(contents)로 반환하는 제네릭 클래스를 생성하였습니다.

## Service

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class NoOffsetService {
 
    private final TodoRepository todoRepository;
 
    // 첫 페이지 가져오기
    public NoOffsetInfoResponse<NoOffsetResponse> getFirstPage(int pageSize) {
        return getPage(null, pageSize, true);
    }
 
    // 다음 페이지 가져오기
    public NoOffsetInfoResponse<NoOffsetResponse> getNextPage(Long lastId, int pageSize) {
        return getPage(lastId, pageSize, false);
    }
    
    // 페이지 가져오기
    public NoOffsetInfoResponse<NoOffsetResponse> getPage(Long lastId, int pageSize, boolean isFirstPage) {
        List<Todo> todos;
        if (isFirstPage) {
            todos = todoRepository.findFirstPage();
        } else {
            todos = todoRepository.findNextPage(lastId);
        }
 
        List<Todo> limitedPage = todos.stream().limit(pageSize).toList(); // size에 따른 limit 설정
        boolean hasData = !todos.isEmpty(); // 현재 페이지에 데이터 존재 여부
        boolean hasNext = todos.size() > pageSize; // 다음 데이터 존재 여부
 
        return NoOffsetInfoResponse.of(
                limitedPage.stream().map(NoOffsetResponse::of).toList(),
                pageSize,
                hasData,
                hasNext
        );
    }
}
```

Service클래스에서 요청받은 size에 따른 limit 설정을 하여 반환합니다.

## Controller

```java
@RestController
@RequiredArgsConstructor
public class NoOffsetController {
 
    private final NoOffsetService noOffsetService;
 
    // 첫 페이지
    @GetMapping("/first")
    public NoOffsetInfoResponse<NoOffsetResponse> getFirstPage(@Valid NoOffsetRequest noOffsetRequest) {
        return noOffsetService.getFirstPage(noOffsetRequest.getSize());
    }
 
    // 다음 페이지
    @GetMapping("/next")
    public NoOffsetInfoResponse<NoOffsetResponse> getNextPage(NoOffsetRequest noOffsetRequest) {
        return noOffsetService.getNextPage(noOffsetRequest.getLastId(), noOffsetRequest.getSize());
    }
}
```

첫 페이지와 다음 페이지 Controller 메서드를 분리하여 요청에 대한 값을 처리하도록 구현하였습니다.

<img width="490" alt="7" src="https://github.com/user-attachments/assets/d6e10bd9-29fd-4990-838c-b8fcda101c9f" />
<img width="503" alt="8" src="https://github.com/user-attachments/assets/7f02962a-d0e9-4d52-b134-862b38bdb48b" />

첫 페이지 호출 시 size 설정 값에 따라 응답하는 모습을 볼 수 있습니다. (size 입력 없이 호출 시 앞서 설정해 놓은 default size 응답)

<img width="506" alt="9" src="https://github.com/user-attachments/assets/13524745-ab15-41ea-8401-95469e3558d1" />

lastId를 설정하여 요청 시 다음 페이지를 응답하는 모습을 볼 수 있습니다.

# JPA + QueryDSL no offset Pagination 예제코드

QueryDSL을 사용한 no offset Pagination 예제코드에서는 메서드를 분리하지 않고 하나로 사용하여 BooleanExpresstion을 사용한 동적 쿼리로 작성하여 lastId를 nullable 할 수 있게 하였습니다.   
JPQL로 구현한 예제코드에서 응답 JSON, Entity, Request, Response는 동일하며 외에 변경되는 로직 중점으로 작성하도록 하겠습니다.

## Config

```java
@Configuration
public class QueryDslConfig {
 
    @PersistenceContext
    private EntityManager entityManager;
 
    @Bean
    public JPAQueryFactory queryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

offset 예제코드와 마찬가지로 QueryDSL을 사용하기 위한 Config 클래스를 생성합니다.

## QueryRepository

```java
@Repository
@RequiredArgsConstructor
public class QueryRepository {
 
    private final JPAQueryFactory jpaQueryFactory;
 
    public List<Todo> findPage(Long lastId, int pageSize) {
        return jpaQueryFactory
                .selectFrom(todo)
                .where(lastIdCondition(lastId)) // lastId보다 작은 id 조건
                .orderBy(todo.id.desc())
                .limit(pageSize)
                .fetch();
    }
 
    private BooleanExpression lastIdCondition(Long lastId) {
        return lastId != null ? todo.id.lt(lastId) : null;
    }
}
```

BooleanExpression 메서드를 생성하여 lastId에 대한 값을 nullable 할 수 있게 동적쿼리를 작성하였습니다.

## Service

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class NoOffsetService {

    private final QueryRepository queryRepository;

    public NoOffsetInfoResponse<NoOffsetResponse> getPage(Long lastId, int pageSize) {
        List<Todo> todos = queryRepository.findPage(lastId, pageSize);

        boolean hasData = !todos.isEmpty();
        boolean hasNext = todos.size() == pageSize;

        return NoOffsetInfoResponse.of(
                todos.stream().map(NoOffsetResponse::of).toList(),
                pageSize,
                hasData,
                hasNext
        );
    }
}
```

Service 클래스에서 JPQL 구현예제와의 변경점은 first페이지와 next페이지를 분리하지 않고 하나의 메서드로 생성하였습니다.

## Controller

```java
@RestController
@RequiredArgsConstructor
public class NoOffsetController {
 
    private final NoOffsetService noOffsetService;
 
    @GetMapping("/noOffset")
    public NoOffsetInfoResponse<NoOffsetResponse> getNextPage(NoOffsetRequest noOffsetRequest) {
        return noOffsetService.getPage(noOffsetRequest.getLastId(), noOffsetRequest.getSize());
    }
}
```

Controller 클래스 또한 마찬가지로 first페이지와 next페이지를 분리하지 않고 하나의 메서드로 생성하였습니다.

QueryDSL을 통한 예제코드 역시 JPQL의 예제코드와 동일한 결과를 보여줍니다.

## no offset 정리

no offset 페이지네이션은 대량의 데이터를 처리할 때 성능을 최적화하기 위한 페이징 방식입니다.   
OFFSET을 사용하지 않고 WHERE 절을 이용해 특정 기준을 설정한 후 데이터를 조회합니다. (마지막 데이터의 ID나 그 외 다른 기준을 설정한 후 데이터를 조회합니다.   
이는 데이터 조회 시 offset과는 달리 매번 로드시마다 처음 조회를 하는 것과 같은 성능을 보여주며 다량의 데이터에 적합합니다.   
하지만 사용자가 원하는 페이지를 특정하여 조회할 수 없다는 단점이 있습니다.

# 테스트코드를 통한 offset과 no offset의 성능차이 알아보기

테스트코드를 통해 offset과 no offset의 성능차이를 알아보겠습니다.

테스트환경은 다음과 같습니다.   
- M3 macOS
- H2 DataBase (In-Memory)
- 테스트 실행 전 10만 개의 데이터 삽입

## offset 테스트 코드

```java
@SpringBootTest
public class OffsetPaginationTest {
 
    @Autowired
    private OffsetService offsetService;
    @Autowired
    private TodoRepository todoRepository;
 
    @BeforeEach
    public void setUp() {
        todoRepository.deleteAllInBatch();
        for (int i = 0; i < 100000; i++) {
            Todo todo = Todo.test("Todo " + i, "Content " + i, TodoStatus.TODO);
            todoRepository.save(todo);
        }
    }
 
    @Test
    public void offsetFirstPaginationTest() {
        // given
        int iterations = 10;
        long totalDuration = 0;
 
        // when
        for (int i = 0; i < iterations; i++) {
            long startTime = System.currentTimeMillis();
            offsetService.offsetPagination(1, 10);
            long endTime = System.currentTimeMillis();
            totalDuration += (endTime - startTime);
        }
 
        // then
        long averageDuration = totalDuration / iterations;
        System.out.println("Offset Pagination 첫 페이지 평균 테스트 결과: " + averageDuration + " ms");
    }
 
    @Test
    public void offsetLastPaginationTest() {
        // given
        int iterations = 10;
        long totalDuration = 0;
 
        // when
        for (int i = 0; i < iterations; i++) {
            long startTime = System.currentTimeMillis();
            offsetService.offsetPagination(10000, 10);
            long endTime = System.currentTimeMillis();
            totalDuration += (endTime - startTime);
        }
 
        // then
        long averageDuration = totalDuration / iterations;
        System.out.println("Offset Pagination 마지막 페이지 평균 테스트 결과: " + averageDuration + " ms");
    }
}
```

setUp 메서드를 통해 테스트 실행 전에 10만 개의 데이터를 생성한 후, 첫 페이지와 마지막 페이지 조회 성능을 각각 10회 반복하여 평균값을 측정하였습니다.

<img width="573" alt="10" src="https://github.com/user-attachments/assets/199cbcbb-11ba-42db-9e8a-5135040ca09a" />
<img width="597" alt="11" src="https://github.com/user-attachments/assets/222ccc25-fa38-4970-9771-5e1a15c414fe" />

- 첫 페이지 조회 성능 : 평균 6ms   
- 마지막 페이지 조회 성능 : 평균 12ms   
테스트 결과 첫 페이지와 마지막 페이지 조회 성능에서 약 2배의 시간 차이가 발생하였습니다.   
이는 offset 기반의 페이징에서 조회하는 데이터의 양이 증가할수록 성능이 점점 저하된다는 것을 의미합니다.

## no offset 테스트 코드

```java
@SpringBootTest
public class NoOffsetPaginationTest {
 
    @Autowired
    private NoOffsetService noOffsetService;
    @Autowired
    private TodoRepository todoRepository;
 
    @BeforeEach
    public void setUp() {
        todoRepository.deleteAllInBatch();
        // 데이터 초기화
        for (int i = 0; i < 100000; i++) {
            Todo todo = Todo.test("Todo " + i, "Content " + i, TodoStatus.TODO);
            // 추가적인 필드 세팅
            todoRepository.save(todo);
        }
    }
 
    @Test
    public void NoOffsetFirstPaginationTest() {
        // given
        int iterations = 10;
        long totalDuration = 0;
 
        // when
        for (int i = 0; i < iterations; i++) {
            long startTime = System.currentTimeMillis();
            noOffsetService.getPage(1L, 10);
            long endTime = System.currentTimeMillis();
            totalDuration += (endTime - startTime);
        }
 
        // then
        long averageDuration = totalDuration / iterations;
        System.out.println("No Offset Pagination 첫 페이지 평균 테스트 결과: " + averageDuration + " ms");
    }
 
    @Test
    public void NoOffsetLastPaginationTest() {
        // given
        int iterations = 10;
        long totalDuration = 0;
 
        // when
        for (int i = 0; i < iterations; i++) {
            long startTime = System.currentTimeMillis();
            noOffsetService.getPage(10000L, 10);
            long endTime = System.currentTimeMillis();
            totalDuration += (endTime - startTime);
        }
 
        // then
        long averageDuration = totalDuration / iterations;
        System.out.println("No Offset Pagination 마지막 페이지 평균 테스트 결과: " + averageDuration + " ms");
    }
}
```

offset 테스트때와 마찬가지로 setUp메서드를 통해 10만 개의 데이터를 생성하고 동일하게 첫 페이지와 마지막 페이지 조회 성능을 10회 반복 측정하였습니다.

<img width="592" alt="12" src="https://github.com/user-attachments/assets/14475b9e-9831-4fc1-be3b-5d7ccfc8f645" />
<img width="611" alt="13" src="https://github.com/user-attachments/assets/dc028c45-9996-47a9-b443-79390f9c6ac5" />

- 첫 페이지 조회 성능 : 평균 5ms   
- 마지막 페이지 조회 성능: 평균 7ms   
no offset 방식의 경우 첫 페이지와 마지막 페이지의 조회 성능차이가 거의 없으며, 이는 다량의 데이터 조회 시 offset 방식보다 no offset 방식이 더욱 효율적임을 알 수 있습니다.

offset 방식에서는 첫 페이지와 마지막 페이지 조회 시 성능차이가 약 2배 발생하였고 no offset 방식에서는 첫 페이지와 마지막 페이지 조회 성능에 차이가 거의 없었습니다.   
결론적으로 다량의 데이터를 다룰 시 offset 방식보다 no offset 방식이 더욱 유리할 수 있음을 알 수 있었습니다.

# 정리

## offset pagination

- SQL에서 LIMIT와 OFFSET을 사용하여 페이지 번호에 따라 데이터를 가져오며, 특정 위치의 데이터부터 조회하여 원하는 페이지를 반환하는 방식입니다.
- 장점
  - 간단한 구현이 가능하며 SpringDataJPA, 기본 SQL에서 쉽게 구현이 가능합니다.
  - 페이지 번호를 통해 사용자가 원하는 특정 페이지로 이동이 가능합니다.
- 단점
  - 데이터가 많을수록 성능이 저하됩니다. OFFSET의 값이 클수록 데이터베이스는 모든 데이터를 처음부터 스캔하여 성능이 점점 저하될 수 있습니다.
  - 데이터가 추가되거나 삭제될 경우, 페이지가 밀리는 현상이 발생할 수 있습니다.
- offset 방식의 경우 다량의 데이터를 다루지 않거나, 간단한 페이지네이션이 필요한 경우에 적합하며, 데이터가 비교적 적고 페이지를 통해 특정 페이지로 이동하는 기능이 중요한 경우. 예를 들어, 게시판이나 간단한 리스트형 웹 서비스에서 유용합니다.

## no offset pagination

- 이전 페이지의 마지막 데이터를 기준으로 다음 데이터를 가져오며, OFFSET을 사용하지 않고 WHERE 조건으로 특정 기준을 기준으로 데이터를 조회합니다.
- 장점
  - OFFSET을 사용하지 않고 조회된 데이터의 마지막 특정기준으로 다음 페이지를 가져오기 때문에 다량의 데이터를 처리할 때도 성능의 저하가 적습니다.
  - 데이터가 추가되거나 삭제되더라도 WHERE 조건에 의해 특정 기준을 기준으로 계속해서 데이터를 조회하여 데이터가 밀리는 현상이 적습니다.
- 단점
  - 마지막으로 조회된 데이터의 ID나 다른 특정 기준값을 클라이언트와 서버 간에 유지해야 하여 구현이 복잡할 수 있습니다.
  - 페이지 번호를 통해 사용자가 특정페이지로 이동하는 기능에 제한적입니다.
- no offset 방식의 경우 대용량 데이터를 효율적으로 처리해야 하는 경우. 예를 들어, 로그 데이터, 실시간 피드 등에서 성능 최적화가 필요할 때 사용됩니다.

페이지네이션에 대해 알게 되며 여기저기 많이 찾아보고 읽어보고 구현하며 학습하던 중 offset 방식보다 no offset 방식이 성능적으로 매우 우수하다는 내용을 많이 접했습니다.   
그럼 성능적으로 우수한 no offset 방식을 항상 써야 하나?라는 의문이 생겼는데 각 방식에는 그에 대한 사용성이 분명히 있었고 그 차이점에 대해 학습하여 그 내용을 토대로 글을 작성하게 되었습니다.
