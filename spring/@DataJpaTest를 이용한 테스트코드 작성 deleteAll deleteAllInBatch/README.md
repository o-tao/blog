# @DataJpaTest
@DataJpaTest 어노테이션은 JPA관련 테스트 설정만 로드합니다.
DataSource의 설정이 정상적인지, JPA를 이용하여 데이터를 저장, 수정, 삭제하는지 등의 테스트가 가능하며, 내장형 데이터베이스를 사용하여 실제 데이터베이스를 사용하지 않고 테스트 데이터베이스로 테스트가 가능합니다.
(H2, DERBY, HSQLDB)

# 테스트코드 작성
- SpringBoot 3.3.3
- Java21
- Junit5
- Spring Data JPA

## 테스트코드 환경 구축

### Entity
```java
@Entity
@Getter
@NoArgsConstructor
public class Board {
	
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String title;
    
    private String content;
    
    private Board(String title, String content) {
    	this.title = title;
        this.content = content;
    }
    
    public static Board create(String title, String content) {
    	return new Board(title, content);
    }
    
}
```

Test 코드에서 값을 받아올 생성자를 추가하였습니다.

### Repository

```java
public interface BoardRepository extends JpaRepository<Board, Long> {
    
}
```

### Test를 위한 application.yml
```yaml
server:
  port: 80

spring:
  datasource:
    url: jdbc:h2:~/test
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        show_sql: true
        format_sql: true

logging:
  level:
    org:
      hibernate:
        sql: debug
        type:
          descriptor:
            sql:
              BasicBinder: trace
```

@DataJpaTest를 이용한 Test시 yml파일에 사용 데이터베이스를 직접 입력하지 않고
```java
testRuntimeOnly 'com.h2database:h2'
```

라이브러리만 build.gradle에 추가해도 무관합니다.   
(필자는 Test를 위한 yml설정을 어떻게 하였는지 직관적으로 확인하기 위해 작성)

![Alt text](/spring/@DataJpaTest를 이용한 테스트코드 작성 deleteAll deleteAllInBatch/images/1.png)


main/resources 아래 application.yml   
test/resources 아래 application.yml   
이렇게 구분 지어 생성하면 각 main실행, test실행별로 application.yml을 나누어 적용이 가능해집니다.

---

## Test 코드
```java
@DataJpaTest
class BoardRepositoryTest {
	
    @Autowired
    private BoardRepository boardRepository;
    
    @AfterEach
    public void clear() {
    	boardRepository.deleteAllInBatch(); // 1
    }
    
    @Test
    @DisplayName("board에 저장한 값을 조회할 수 있다")
    public void saveTest() {
    	 // given
        
        // title, content 내용을 매개변수로 전달
        Board board = Board.create("todo-list", "hello");
        
        // when
        
        // 매개변수로 전달된 내용 save
        boardRepository.save(board);
        
        // save가 되었는지 확인하기 위한 조회
        Board findBoard = boardRepository.findAll().stream().findAny().orElseThrow();
        
        // then
        
        // 조회된 값과 저장된 값이 동일한지 확인
        Assertions.assertThat(findBoard.getTitle()).isEqualTo(board.getTitle());
        Assertions.assertThat(findBoard.getContent()).isEqualTo(board.getContent());
        
    }

}
```

1. 테스트 실행 후 JPA 데이터 삭제
deleteAll이 아닌 deleteAllInBatch 사용 이유 :
```java
// deleteAll을 사용하였을 경우 동작 쿼리를 확인하기 위한 임시 Test 코드

@Autowired
private BoardRpository boardRepository;

@Test
public void deleteAll() {
	// given
    Board board1 = Board.create("title1", "content1");
    Board board2 = Board.create("title2", "content2");
    Board board3 = Board.create("title3", "content3");
    
    // when
    boardRepository.save(board1);
    boardRepository.save(board2);
    boardRepository.save(board3);
    
    // then
    boardRepository.deleteAll();
}
```

![Alt text](/spring/@DataJpaTest를 이용한 테스트코드 작성 deleteAll deleteAllInBatch/images/2.png)
![Alt text](/spring/@DataJpaTest를 이용한 테스트코드 작성 deleteAll deleteAllInBatch/images/3.png)

deleteAll을 이용하여 Test실행 결과 JPA데이터를 삭제하기 위해 조회 후 각 PK별로 delete 쿼리가 동작하여 처리되는 것을 확인할 수 있습니다.

```java
// deleteAllInBatch를 사용하였을 경우 동작 쿼리를 확인하기 위한 임시 Test 코드

@Autowired
private BoardRpository boardRepository;

@Test
public void deleteAll() {
	// given
    Board board1 = Board.create("title1", "content1");
    Board board2 = Board.create("title2", "content2");
    Board board3 = Board.create("title3", "content3");
    
    // when
    boardRepository.save(board1);
    boardRepository.save(board2);
    boardRepository.save(board3);
    
    // then
    boardRepository.deleteAllInBatch(); // deleteAll -> deleteAllInBatch 변경
}
```

![Alt text](/spring/@DataJpaTest를 이용한 테스트코드 작성 deleteAll deleteAllInBatch/images/4.png)

deleteAllInBatch를 이용하여 Test실행 결과 JPA데이터 삭제 시 한 번의 delete 쿼리가 동작하여 처리되는 것을 확인할 수 있습니다.

Test시 각 PK별로 delete쿼리를 동작하기보다 한 번에 처리하여 불필요한 데이터사용을 막을 수 있기 때문에 deleteAll이 아닌 deleteAllInBatch를 사용하였습니다.
