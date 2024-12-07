데이터 생성, 수정 등을 Postman을 통해 테스트하던 중 Request와 Response과정에서 에러가 발생하였습니다.

# Request Error

`SQLIntegrityConstraintViolationException`

# Response Error

`HttpMediaTypeNotAcceptableException`

Request, Response 두 에러가 발생하였는데 에러내용으로 확인했을 때 많은 이유가 존재하지만 필자의 경우 결국 직렬화, 역직렬화 과정에서 null 값으로 처리되어 발생한 에러였습니다.

이번 포스팅에서는 에러가 발생한 원인인 null 값으로 처리되는 이유와 해결방법에 대해 알아보겠습니다.

우선 null 값으로 처리되는 이유와 해결방법에 대해 알아보기 전, JSON과 직렬화, 역직렬화에 대해 간단히 알아보겠습니다.

# JSON 이란?

JavaScript 객체 문법으로 구조화된 데이터 교환 형식입니다.   
Java나 JavaScript, python과 같은 여러 언어에서 데이터 교환 형식으로 쓰이며, 단순 배열, 문자열로도 표현이 가능합니다.

JSON은 `Key-Value` 쌍으로 데이터를 표현하며, Key는 항상 문자열이어야 하지만, Value는 문자열, 숫자, 객체 등 다양한 형태로 올 수 있습니다.

```json
// ex 숫자, 문자열, 객체
{
  "id": 1, // 숫자
  "title": "title 내용", // 문자열
  "content": "content 내용",
  "member": { // 객체
    "id": 1,
    "email": "tao@example.com"
  }
}

```

```json
// ex 배열
{
  "contents": [
    {
      "title": "title 내용1",
      "content": "content 내용1",
    },
    {
      "title": "title 내용2",
      "content": "content 내용2",
    },
    {
      "title": "title 내용3",
      "content": "content 내용3",
    }
  ]
}

```

## JSON 직렬화(Serialization)

JSON 직렬화란 애플리케이션에서 사용하는 객체나 자료구조를 JSON 형식의 텍스트 데이터로 변환하는 과정입니다.      
직렬화는 시스템 간 데이터 전송, 데이터 저장, 애플리케이션 내에서의 데이터 처리에서 널리 사용되는 기술입니다. 특히 JSON은 경량 텍스트 포맷으로, 다양한 시스템에서 호환 가능하여 웹, 모바일, 서버 개발 등 다양한 영역에서 데이터 교환 형식으로 사용됩니다.

컴퓨터 메모리 상에 존재하는 객체(Object)를 문자열(String)로 변환하는 것을 의미합니다.

## JSON 역직렬화(Deserialization)

JSON 역직렬화란 직렬화된 데이터를 다시 원래의 객체나 자료구조로 복원하는 과정입니다.   
JSON이나 XML 같은 텍스트 기반 형식으로 저장된 데이터를 읽어, 코드 내에서 사용 가능한 객체로 변환하는 것을 말하며, 이를 통해 네트워크로 전송되거나 파일로 저장된 데이터를 다양한 로직에 사용할 수 있습니다.

문자열(String)을 컴퓨터 메모리 상에 존재하는 객체(Object로) 변환하는 것을 의미합니다.

---

# Request, Response 중 null 값으로 처리된 이유

`Request 에러`에서는 역직렬화 과정에서 난 에러로 null 값으로 요청되어 RequestBody로 요청받은 값을 null로 가져와 NotNull 설정해 둔 데이터베이스 컬럼에 저장하려고 해 발생한 에러였고,   
`Response 에러`에서는 데이터베이스에는 저장이 되었으나 Response 응답으로 허용되는 표현이 없어 생긴 에러였습니다.

Request, Response DTO 클래스에서는 일반적으로 private으로 선언되는데, private으로 선언된 해당 필드가 클래스외부에서 직접 접근할 수 없도록 제한하여 직렬화, 역직렬화 과정에서 null 값으로 처리된 것입니다.

```java
// Request DTO private 필드
public class MemberRequest {

    private String email;
    private String password;
}

// Response DTO private 필드
public class MemberResponse {

    private Long id;
    private String email;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

```


해당 문제에 대해서는 public으로도 해결되는 문제지만 캡슐화 원칙이 위반되며 데이터 무결성을 해칠 수 있으므로 권장하지 않습니다.

필자의 경우는 아니었지만 Reqeuest, Response 중 null 값으로 처리되는 것에는 여러 가지 이유가 존재합니다.

- JSON 형식 불일치
```json
// 클라이언트 요청 JSON
{
  "user_email": "tao@example.com",
  "password": "1234"
}

```
```java
// DTO 예시
public class MemberRequest {
    private String email; // 여기서 'email'이 아니라 'user_email'이 전송됨
    private String password;
}

```

- 데이터 변환 오류 (타입이 맞지 않는 경우)
```json
// 클라이언트 요청 JSON
{
  "email": "tao@example.com",
  "password": 1234 // 문자열이 아닌 숫자
}

```

- 불필요한 필드 생략 (필수 필드 누락)
```json
// 클라이언트 요청 JSON
{
  "email": "tao@example.com"
  // password 필드 누락
}

```

등 여러 가지 이유로 인해 null 값으로 처리될 수 있습니다.

# 해결방법

## Request DTO에 Setter 설정

@Setter어노테이션 또는 Setter메서드가 있는 경우 메서드를 통해 외부에서 필드에 값을 할당할 수 있게 됩니다.   
Lombok의 경우 컴파일 시점에 자동으로 메서드를 생성합니다.

```java
// Setter 메서드 생성

public class MemberRequest {

    private String email;
    private String password;
    
    public void setEmail(String email) {
    	this.email = email;
    }
    
    public void setPassword(String password) {
    	this.password = password;
    }
}

```

```java
// Lombok 어노테이션 설정

@Setter
public class MemberRequest {

    private String email;
    private String password;
}

```

Spring의 데이터 바인딩 과정에서 JSON 요청(Request) 데이터가 DTO로 변환될 때, JSON의 키와 DTO의 필드가 매핑되면서 해당 Setter 메서드가 호출됩니다.

```json
// 요청 ex
{
  "email": "tao@exemple.com",
  "password": "1234"
}

```

해당 요청이 들어오면 Setter 메서드가 호출되어 필드에 값이 설정되어 null 값으로 처리되는 에러를 피할 수 있습니다.

## Response DTO에 Getter 설정

@Getter어노테이션 또는 Getter메서드가 있는 경우 메서드를 통해 외부에서 필드에 값을 읽을 수 있게 됩니다.   
Lombok의 경우 컴파일 시점에 자동으로 메서드를 생성합니다.

```java
// Getter 메서드 생성

public class MemberResponse {

    private Long id;
    private String email;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    public Long getId() {
        return id;
    }

    public String getEmail() {
        return email;
    }

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public LocalDateTime getUpdatedAt() {
        return updatedAt;
    }
}

```

```java
// Lombok 어노테이션 설정

@Getter
public class MemberResponse {

    private Long id;
    private String email;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

```

Spring이 DTO 객체를 JSON으로 변환(직렬화)할 때, 각 필드의 값을 가져오기 위해 Getter 메서드를 호출합니다.

## JSON 어노테이션 사용

JSON 어노테이션은 JOSN 직렬화 및 역직렬화 과정에서 Spring과 Jackson이 데이터를 올바르게 매핑하고 변환할 수 있도록 하는 데 사용됩니다.

- @JsonProperty
각 필드에 선언하여 매핑할 JSON의 Key를 문자열로 적어줍니다. Response DTO에도 적용이 가능합니다.

```java
// Request DTO

public class MemberRequest {
    @JsonProperty("email")
    private String email;
    
    @JsonProperty("password")
    private String password;
    
}
```

```json
// 매핑되는 JSON `Key-value`
{
  "email": "tao@example.com",
  "password": "1234"
}

```

만약 JSON의 Key와 필드의 이름을 다르게 설정할 경우 문자열로 적어주는 Key를 변경해 줍니다.

```java
// Request DTO

public class MemberRequest {
    @JsonProperty("user_email") // JSON의 "user_email" Key로 매핑
    private String email;
    
    @JsonProperty("user_password") // JSON의 "user_password" Key로 매핑
    private String password;
}

```

```json
// 매핑되는 JSON `Key-value`
{
  "user_email": "tao@example.com",
  "user_password": "1234"
}

```

# 정리

API 요청 및 응답 과정에서 발생할 수 있는 null 값 처리 문제를 중심으로 다양한 원인과 해결 방법에 대해 알아보았습니다.

- Request 과정에서 역직렬화 시 null 값으로 처리된 데이터가 NotNull 제약 조건이 있는 데이ㅓ베이스 컬럼에서 저장하려 할 때 발생한 `SQLIntegrityConstraintViolationException`
- Response 과정에서는 데이터베이스에 저장은 되었지만, 적절한 표현이 없어 발생한 `HttpMediaTypeNotAcceptableException`

결국 API 요청 및 응답 과정에서 null 값으로 처리되어 발생한 문제였고 이를 해결하기 위해 단순히 Getter, Setter를 사용하거나 Json의 어노테이션을 선언하여 해결할 수 있었지만 "왜 해당 어노테이션을 선언했을 때 발생한 에러가 해결됐을까?" 라는 궁금증이 생겨 글로 작성하게 되었습니다.   
에러가 발생한 이유에 대해 이해하기 위해 JSON의 직렬화, 역직렬화에 대해 이해가 필요하였고, null값으로 처리되는 데에는 정말 많은 이유가 존재한다는 것을 알게 되었습니다.
