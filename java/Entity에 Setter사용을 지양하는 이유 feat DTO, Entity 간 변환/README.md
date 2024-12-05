JPA학습을 하다 보면 한 번쯤은 Entity에 Setter사용을 지양하자 라는 내용을 들어보았을 것입니다.   
이번 포스팅에서는 Entity에 Setter사용을 지양하는 이유와 Setter를 사용하지 않고 데이터 저장 및 수정, DTO, Entity 간 변환에 대하여 알아보겠습니다.

# Entity에 Setter사용을 지양하는 이유

Entity의 경우 비즈니스 로직이 존재하고 실제 데이터가 변경되는데 그 과정에서 Setter를 사용하여 값을 설정할 경우 몇 가지 불리한 점이 존재합니다.

## 사용자의 의도파악이 불분명하다

```java
Member member = new Member();
member.setEmail("tao@exemple.com");
member.setName("오팔봉");

```

Setter를 사용해 값을 설정할 경우 생성하는 것인지, 변경하는 것인지 그 의도파악이 어렵습니다.   
이는 코드의 복잡성이 증가할수록 더욱 한눈에 알기 힘들어질 수 있습니다.

## 일관성을 유지하기 어렵다

```java
public User updateUser(Long id) {
    User user = findById(id);
    user.setEmail("bong@exemple.com");
    user.setName("tao");
    user.setRole("USER");
    return user;
}

```

Setter를 사용하게 되면 외부에서 임의로 객체의 상태를 변경할 수 있어, 코드의 명확성을 저하시킬 수 있습니다.   
특히 객체의 여러 속성이 서로 연관되어 있을 때 하나의 속성을 변경하게 되면 다른 속성도 함께 변경되어야 할 수 있는데 Setter를 통해 임의로 속성을 변경할 경우 의존성을 고려하지 못해 일관성을 유지하는 데에 불리할 수 있습니다.   
이러한 부분에서 예상하지 못한 버그가 발생할 수 있습니다.

## 유지보수가 어렵다

Setter를 사용해 값을 설정할 경우 앞서 얘기한 것처럼 의도파악 및 일관성 유지가 어렵습니다. 이는 어느 부분에서 값을 설정하였는지 변경 포인트를 추적하기 어려워 유지보수 측면에서도 불리하게 작용합니다.

---   

# Setter 없이 데이터를 저장 및 수정 방법, DTO와 Entity 간 변환

Setter의 경우 JPA의 Transaction 안에서 Entity의 변경사항을 감지하여 Update 쿼리를 생성합니다.   
즉 Setter메서드는 update 기능을 수행합니다. 여러 곳에서 Entity를 생성하여 Setter를 통해 Update를 수행한다면 코드의 복잡성이 증가할수록 해당 변경포인트를 추적 하기가 매우 힘들어집니다.

회원에 대한 예시코드를 통하여 Setter를 사용하지 않고 데이터를 저장, 수정하는 방법에 대하여 알아보도록 하겠습니다.

## Entity

```java
@Entity
@Getter
@Table(name = "members") // 테이블 이름 설정
@NoArgsConstructor
public class Member extends BaseEntity { // BaseEntity를 통해 중복필드를 상속받음

    @Column(nullable = false, length = 100) // NotNull, 문자허용길이=100
    private String email;

    @Column(nullable = false, length = 60) // NotNull, 문자허용길이=60
    private String password;

    public Member(String email, String password) { // 생성자
        this.email = email;
        this.password = password;
    }

    public void update(String email, String password) { // update 메서드
        this.email = email;
        this.password = password;
    }
}

```

```java
@Getter
@MappedSuperclass
public class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreationTimestamp // 생성시간 관리
    @Column(name = "created_at", nullable = false, updatable = false) // 컬럼이름, NotNull, 업데이트 불가
    private LocalDateTime createdAt;

    @UpdateTimestamp // 수정시간 관리
    @Column(name = "updated_at", nullable = false) // 컬럼이름, NotNull
    private LocalDateTime updatedAt;
}

```

중복되는 필드를 BaseEntity로부터 상속받아 Member Entity를 생성하였습니다.   

BaseEntity생성에 대한 참고자료입니다. [SpringDataJPA 공통필드 BaseEntity로 관리하기](https://tao-tech.tistory.com/14)

## Repository

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    boolean existsByEmail(String email); // 중복회원 검증
}

```

중복되는 아이디를 검증하기 위한 쿼리 메서드를 생성하였습니다.

## Request DTO
```java
@Setter
@NoArgsConstructor
public class MemberRequest {

    // 이메일 형식 패턴
    @Pattern(regexp = "^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,6}$", message = "이메일 형식이 올바르지 않습니다.")
    @NotBlank(message = "이메일을 입력하세요.") // null, "", " "(공백) 모두 비허용
    private String email;

    @NotBlank(message = "비밀번호를 입력하세요.") // null, "", " "(공백) 모두 비허용
    private String password;

    public Member toEntity() { // Request값 toEntity화
        return new Member(email, password);
    }
}

```

요청받는 값을 추가하여 MemberRequest DTO를 생성하였습니다.   
Request받은 값을 toEntity 메서드를 통해 Member Entity 생성자에게 값을 전달합니다.

이때, MemberRequest에 @Setter를 생성하였는데 Entity는 Setter를 지양하는데 왜 DTO에서는 Setter를 사용하지? 라는 의문이 들 수 있습니다.

Entity사용 시에는 비즈니스 로직이 존재하고 실제 데이터가 변경되는데 DTO에서는 단지 데이터의 전달 목적으로만 사용되어 DTO에는 Setter를 사용해도 무방합니다.   
필자는 Request DTO에 Setter를 생성하였지만 실제 로직에서 데이터 전달을 위해 toEntity 메서드를 활용하고 테스트코드 작성시에 임시 데이터값을 설정하기위해서만 사용하고 있습니다.

추가로 Pattern, NotBlank 등의 기능을 사용하기 위해서는 Validation 의존성을 추가해야합니다.

```bash
implementation 'org.springframework.boot:spring-boot-starter-validation'

```

## Response DTO

```java
@Builder
@Getter
public class MemberResponse {

    private Long id;
    private String email;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss") // DateTime 패턴 설정
    private LocalDateTime createdAt;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss") // DateTime 패턴 설정
    private LocalDateTime updatedAt;

    public static MemberResponse of(Member member) {
        return MemberResponse.builder()
                .id(member.getId())
                .email(member.getEmail())
                .createdAt(member.getCreatedAt())
                .updatedAt(member.getUpdatedAt())
                .build();
    }
}

```
응답하기 위한 필요값을 추가하여 Response DTO를 생성하였습니다.

## Controller

```java
@RestController
@RequestMapping("/api/members")
@RequiredArgsConstructor
public class MemberController {

    private final MemberService memberService;

    @PostMapping
    public MemberResponse create(@RequestBody @Valid MemberRequest memberRequest) {
        Member member = memberService.create(memberRequest.toEntity());
        return MemberResponse.of(member);
    }

    @PutMapping("/{id}")
    public MemberResponse update(@PathVariable("id") Long id, @RequestBody @Valid MemberRequest memberRequest) {
        Member member = memberService.update(id, memberRequest.toEntity());
        return MemberResponse.of(member);
    }

}

```
현재 create와 update 각 메서드에서 Request 값을 toEntity로 전달하는데 여기서 DTO와 Entity간의 변환이 일어납니다.   
앞서 생성한 Request DTO에 toEntity의 메서드를 생성하였는데, Request 받은 값을 toEntity로 전달하여 Member타입의 toEntity 메서드가 그 값을 Member Entity의 생성자에 전달합니다.   
최종적으로 Service의 로직을 수행 후 그 값을 Response DTO에 of 메서드로 전달하여 Response값을 전달합니다.

## Service

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    @Transactional
    public Member create(Member member) { // Member 생성
        validateMember(member.getEmail());
        return memberRepository.save(member);
    }

    @Transactional
    public Member update(Long id, Member member) { // Member 수정
        validateMember(member.getEmail());
        Member existingMember = memberRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("회원이 존재하지 않습니다."));
        existingMember.update(member.getEmail(), member.getPassword()); // Setter 작업
        return existingMember;
    }

    public void validateMember(String email) { // 중복회원 검증
        if (memberRepository.existsByEmail(email)) {
            throw new IllegalStateException("이미 존재하는 회원입니다.");
        }
    }

}

```
Controller에서 Request 값을 Entity로 변환하여 매개변수로 받아 생성, 수정 로직을 수행합니다.   
이때, update메서드에서 Setter작업이 이루어지는 부분이 Setter를 사용하지 않고 값을 메서드를 통해 전달하여 데이터 수정이 이루어집니다.

해당 비즈니스 로직에서 Setter작업이 이루어지는 곳을 보면 Setter를 사용했을때와 달리 그 의도성이 명확하게 드러나는 것을 확인할 수 있습니다.

```java
// as-is
existingMember.setEmail(member.getEmail());
existingMember.setName(member.getPassword());

// to-be
existingMember.update(member.getEmail(), member.getPassword()); // Setter 작업

```
# 정리

JPA를 활용한 작업에서 Entity에 Setter 사용을 지양하는 이유와, 이를 대체하기위한 방법에 대해 알아보았습니다.   
의도파악의 불명확성, 일관성 유지의 어려움, 유지보수의 복잡성 등을 개선하기 위해 Member Entity에 메서드를 추가하여 Setter 사용을 지양하고 메서드를 통하여 상태를 변경함으로써 비즈니스 로직의 일관성을 유지하고, 코드의 가독성을 높일 수 있으며 이를 통해 사용자는 의도한 바를 명확히 하고, 이후의 유지보수 시 발생할 수 있는 문제를 최소화할 수 있습니다.   
그리고 DTO와 Entity의 역할을 명확히 구분하고 캡슐화의 원칙을 준수하여 객체의 상태를 내부에서 관리함으로써 외부의 영향으로부터 보호할 수 있으며 이는 객체의 책임을 분리하여 각 클래스가 자신의 상태를 일관되게 유지할 수 있게 합니다.
