# API
#### API(Application Programing Interface)
- Application: 고유한 기능을 가진 모든 소프트웨어를 나타냅니다.
- Programing: 컴퓨터가 작업을 수행하기 위해 따를 수 있는 프로그램이라는 명령어 순차 조합을 구성하는 것을 말합니다.
- Interface: 두 애플리케이션 간의 서비스 상호작용을 나타냅니다. 요청과 응답을 사용하여 두 애플리케이션이 서로 통신하는 방법을 정의합니다.

정리하자면 컴퓨터의 기능을 실행, 어떠한 응용프로글매에서 데이터를 주고받기 위한 방법을 의미합니다.

# REST
#### REST(Representational State Transfer)
- Representational: 자원의 표현
- State: 웹 애플리케이션의 상태
- Transfer: 이 상태의 전송

> 자원의 표현에 의한 상태 전달   
> 자원은 문서, 그림, 데이터 등 소프트웨어가 관리하는 모든 것을 의미하며, 자원의 표현은 그 자원을 표현하기 위한 이름   
> ex) 데이터베이스의 회원 정보 = 자원, 'member' = 자원의 표현

HTTP URI를 통해 자원(Resource)을 명시하고, HTTP Method(POST, GET, PUT, DELETE 등)를 통해 해당 자원에 대한 CRUD 행위를 적용하는 것을 말합니다.

## 특징
- 균일한 인터페이스
  - HTTP 표준에 따라 안드로이드, IOS 플랫폼 특정 언어나 기술에 종속되지 않고 모든 플랫폼에 사용할 수 있으며, URI로 지정한 리소스에 대한 조작이 가능한 아키텍처 스타일을 의미합니다.

- 무상태성
  - 작업을 위한 상태정보를 따로 저장하고 관리하지 않습니다. 세션 정보나 쿠키 정보를 별도로 저장하고 관리하지 않기 때문에 API 서버는 들어오는 요청만을 처리합니다. 이로 인해 서비서의 자유도가 높아지고 서버에서 불필요한 정보를 관리하지 않아 구현이 단순해집니다.

- 캐시 가능
  - HTTP라는 기존 웹 표준을 그대로 사용하여 웹에서 사용하는 기존 인프라를 그대로 활용할 수 있습니다. 따라서 HTTP가 가진 캐싱 기능을 적용할 수 있습니다. 이를 통해 서버 부하를 줄이고 성능을 향상시킬 수 있습니다.

- 자체 표현 구조
  - REST API 메세지만으로 이를 쉽게 이해할 수 있는 자체 표현 구조로 되어있습니다.

- Client - Server 구조
  - REST서버는 API제공, 클라이언트는 사용자 인증이나 컨텍스트(세션, 로그인정보) 등을 직접 관리하는 구조로 각각의 역할이 확실히 구분되어 클라이언트와 서버에서 개발해야 할 내용이 명확해지고 서로 간의 의존성이 줄어들게 됩니다.

- 계층형 구조
  - REST 서버는 다중 계층으로 구성될 수 있으며 보안, 로드 밸런싱, 암호화 계층을 추가해 구조상의 유연성을 둘 수 있고 프록시, 게이트웨이 같은 네트워크 기반의 중간매체를 사용할 수 있게 합니다.

# REST API

## REST API 설계규칙

### URI 규칙

- URI 경로에는 소문자를 사용한다.
```text
✅ http://restapi.tistory.com/tao/post-comments

❌ http://restapi.tistory.com/tao/postComments
```

- 언더바(_) 대신 하이픈(-)을 URI 가독성을 위해 사용한다.
  - 가급적 하이픈의 사용도 최소화하며, 정확한 의미나 표현을 위해 단어의 결합이 불가피한 경우에 사용한다.
```text
✅ http://restapi.tistory.com/tao/post-comments

❌ http://restapi.tistory.com/tao/post_comments
```

- 마지막에 슬래시(/)를 포함하지 않는다.
```text
✅ http://restapi.tistory.com/tao

❌ http://restapi.tistory.com/tao/
```

- 행위는 포함하지 않는다.
  - 행위는 URL대신 Method를 사용하여 전달 (GET, POST. PUT, DELETE 등)
```text
✅ http://restapi.tistory.com/tao/1/posts/1

❌ http://restapi.tistory.com/tao/1/delete-post/1
```

- 파일 확장자는 URI에 포함시키지 않는다.
```text
✅ http://restapi.tistory.com/tao/photo

❌ http://restapi.tistory.com/tao/photo.jpg
```

### HTTP 메서드 활용

- GET: Read(조회)
- POST: Create(생성)
- PUT: Update(전체 수정)
- PATCH: Update(일부 수정)
- DELETE: Delete(정보 삭제)

### POST, PUT, PATCH 차이점
#### POST, PUT
POST는 Create(생성), PUT은 Update(수정)에 매칭되는데, RESTful API는 자원에 대한 행위를 Method로 표현하여 자원에 대한 생성은 POST, 자원에 대한 수정은 PUT이 담당한다고 보통 정의합니다.

POST와 PUT은 크게 멱등성 유무의 차이에 있습니다.   
> 멱등성?   
> N번 수행하더라도 결과가 같음을 의미합니다.

#### POST
요청시 서버가 아직 "식별하지 않은" 새 리소스를 생성(등록)합니다. 단순히 데이터를 생성하거나, 변경하는 것을 넘어서 프로세스를 처리해야 하는 경우 사용됩니다.   
ex) 주문에서 결제완료 → 배송시작 → 배송완료 같은 단순히 값 변경을 넘어 프로세스의 사태가 변경되는 경우   
#### 같은 결과 수행 시 새 리소스를 생성하여 멱등하지 않습니다.

#### PUT
요청시 리소스를 대체합니다. 리소스가 있으면 완전히 대체(값을 덮어 씌움), 리소스가 없으면 생성합니다.   
ex) 폴더안에 새로운 폴더를 만들 때 기존에 있는 폴더면 덮어쓰고 없으면 새폴더를 생성   
#### 클라이언트가 리소스를 식별하여 같은 결과 수행 시 결과가 같아 멱등합니다.

#### PUT, PATCH
다음과 같은 리소스가 있다고 가정하였을 때   
```json
{
  "id": 1,
  "title": "tao",
  "content": "tistory"
}
```
#### PUT
"title"값을 "tao" → "tao-tech"로 수정
```json
PUT /board/1
{
  "title": "tao-tech"
}

HTTP/1.1 200 OK
{
  "title": "tao-tech",
  "content": null
}
```
PUT을 이용하여 수정 시 값을 덮어씌우기 때문에 값을 입력하지 않은 content는 null값으로 처리됩니다.

#### PATCH
위의 PUT예시와 같이   
"title"값을 "tao" → "tao-tech"로 수정   
```json
PATCH /board/1
{
	"title": "tao-tech"
}

HTTP/1.1 200 OK
{
	"title": "tao-tech"
    "content": "tistory"
}
```

PATCH를 이용하여 수정시 일부 값이 수정됩니다. (값을 덮어쓰지 않음)

PATCH는 수정만을 담당하며 리소스의 일부분만 수정할 때 사용   
PUT은 리소스의 모든 속성을 수정하기 위해 사용

### HTTP 응답 상태 코드 활용
1xx:정보 응답 / 2xx:성공 응답 / 3xx:리다이렉트 / 4xx:클라이언트 요청 오류 / 5xx:서버 오류

200: OK, 요청을 정상적으로 처리

201: Created, 성공적으로 생성에 대한 요청을 받았으며 그 결과로 서버가 새 리소스를 작성 (일반적으로 POST 또는 PUT 요청의 응답)

400: Bad Request, 잘못된 문법으로 요청을 보내 서버가 이해할 수 없는 상태

401: Unauthorized, 클라이언트가 인증되지 않았거나, 유효한 인증 정보가 부족하여 요청을 거부

404: Not Found, 클라이언트가 요청한 URI를 찾을 수 없는 상태

500: Internal Server Error, 서버의 문제로 응답할 수 없는 상태 (정확한 문제에 대해 구체적인 설명 불가)

# RESTful API
### RESTful 하다.
RESTful API란 REST의 원리를 따르는 시스템을 의미합니다.   
REST를 사용했다 하여 모두가 RESTful 한 것이 아닌, REST API의 설계 규칙을 올바르게 지킨 시스템을 RESTful 하다고 말합니다.

### RESTful 하지 못하다.
모든 CRUD 기능을 POST로 처리하는 API, URI 규칙을 올바르게 지키지 않은 API   
REST APIdml 설계 규칙을 올바르게 지키지 못한 시스템은 REST API를 사용하였지만 RESTful 하지 못한 시스템이라고 할 수 있습니다.

# 정리
RESTful은 REST의 설계 규칙을 잘 지켜 설계된 API를 RESTful 한 API라고 합니다.   
한마디로 REST의 원리를 잘 따르는 시스템을 RESTful이라고 칭합니다.