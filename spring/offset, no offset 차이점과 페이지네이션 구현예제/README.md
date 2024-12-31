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
