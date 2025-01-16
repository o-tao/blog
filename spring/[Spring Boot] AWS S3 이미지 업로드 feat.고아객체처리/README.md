이미지와 같은 정적 파일을 효율적으로 관리하는 것은 웹 애플리케이션 개발에서 자주 마주하게 됩니다. 이때, 클라우드 기반 저장소인 `AWS S3 (Simple Storage Service)`는 높은 확정성과 안정성을 제공합니다.   
이번 포스팅에서는 Java 기반 Spring Boot 환경에서 `MultipartFile`방식을 활용하여 `AWS S3`에 이미지를 업로드하는 방법에 대해 알아보도록 하겠습니다.

# MultipartFile 업로드 방식 개념

시작하기에 앞서 `MultipartFile 업로드 방식의 개념`에 대해 먼저 알아보도록 하겠습니다.

![14](https://github.com/user-attachments/assets/103a2f5c-00de-477f-8b15-8887abb40243)

`MultipartFile`은 Spring에서 제공하는 인터페이스로, 파일 업로드를 간편하게 처리할 수 있도록 돕는 기능입니다. 이 인터페이스는 업로드된 파일의 `이름, 크기, 내용 등`에 접근할 수 있는 메서드를 제공하며, 파일 업로드 작업에 있어 높은 수준의 추상화와 편의성을 제공합니다.
- 업로드된 파일을 임시 디렉터리에 저장 후, 처리 요청이 끝나면 해당 파일을 자동으로 삭제합니다.
- 파일을 메모리가 아닌 `Servlet Container Disk`에 저장하여 메모리 부담을 줄입니다.
- Spring이 제공하는 설정을 통해 업로드 제한 및 동작 방식을 세부적으로 제어할 수 있습니다.

```yml
spring:
  servlet:
    multipart:
      enabled: true # 멀티파트 업로드 지원 여부 (default: true)
      file-size-threshold: 0B # 메모리 대신 디스크에 저장하는 파일의 최소 크기 (default: 0B)
      location: /Users/tao/Documents # 업로드된 파일이 임시로 저장될 디렉터리 경로
      max-file-size: 100MB # 업로드할 단일 파일의 최대 크기 (default: 1MB)
      max-request-size: 100MB # 업로드 요청 전체의 최대 크기 (default: 10MB)
```

`MultipartFile` 업로드 방식을 사용 시 위와 같은 설정을 추가할 수 있습니다.

- 클라이언트 업로드: 클라이언트가 파일을 업로드하면 `AWS(Tomcat)`가 해당 파일을 설정된 임시 디렉터리 `location`에 저장합니다.
- 임시 파일 관리: 임시 디렉터리에 저장된 파일은 요청 처리가 끝나면 자동으로 삭제됩니다.
- 예외 상황: 업로드 중 배포나 서버 장애가 발생하면, 임시 파일이 삭제되지 않고 남아 있을 수 있습니다. 이 경우 별도의 삭제 작업이 필요하므로 `location` 경로를 직접 설정하여 관리를 용이하게 할 수 있습니다.
- 파일 크기와 메모리 할당: `file-size-threshold` 값을 기준으로 파일의 크기가 결정됩니다.
  - 파일의 크기가 `file-size-threshold` 이하일 경우 파일은 메모리에 직접 할당됩니다. 이 방식은 빠르지만 메모리에 부담을 줄 수 있어 적절한 크기 조정이 필요합니다.
  - 파일 크기가 `file-size-threshold`를 초과할 경우 파일은 지정된 `location` 경로에 저장됩니다. 이후 Spring에서 필요한 경우 해당 파일을 읽어 작업을 수행합니다.

![15](https://github.com/user-attachments/assets/b9d71669-976c-4132-9e0a-ef0d4a8b274d)

# AWS S3 인프라 구축

AWS S3를 활용하기 위해 먼저 기본적인 인프라 구축이 필요합니다. 이를 위해 S3 버킷을 생성하고, Spring Boot 애플리케이션에서 사용할 수 있도록 필요한 권한 및 설정을 구성해야 합니다.

<img width="321" alt="1" src="https://github.com/user-attachments/assets/5f83c647-15da-4133-a234-152d5d8c0533" />

[AWS 사이트](https://aws.amazon.com/ko/free/?trk=fa2d6ba3-df80-4d24-a453-bf30ad163af9&sc_channel=ps&ef_id=Cj0KCQiAgJa6BhCOARIsAMiL7V_eq6cvE-4f-C-M0XKUU89B3CZ4ZvrE_FIuRI7JRurftFFdYPt6riwaAnIUEALw_wcB:G:s&s_kwcid=AL!4422!3!563761819834!e!!g!!aws!15286221779!129400439466&gclid=Cj0KCQiAgJa6BhCOARIsAMiL7V_eq6cvE-4f-C-M0XKUU89B3CZ4ZvrE_FIuRI7JRurftFFdYPt6riwaAnIUEALw_wcB&all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all)에 접속 후 로그인 한 뒤 우측 상단에 위치한 리전을 서울로 설정합니다.

## S3 버킷 생성

<div>
<img width="1015" alt="2" src="https://github.com/user-attachments/assets/2c0de54d-b692-488b-b6f7-efa80536c6d5" />   
<br>
<img width="1615" alt="3" src="https://github.com/user-attachments/assets/5a1bf05d-ee2a-4d0f-97ae-f9185a9972e9" />
</div>

S3를 검색하여 이동 후 버킷 만들기를 클릭합니다.   

<div>
<img width="924" alt="4" src="https://github.com/user-attachments/assets/237fce01-e4bf-4d1f-9052-8fa509065f98" />
<br>
<img width="1655" alt="5" src="https://github.com/user-attachments/assets/a86d993e-5742-4b45-8760-f9459df7ef97" />
<br>
<img width="1653" alt="6" src="https://github.com/user-attachments/assets/b6aa6263-328c-4a02-8dad-e0f169fc1e57" />
</div>

버킷 이름 입력 → ACL 활성화 → 퍼블릭 액세스 차단 설정 해제   
위 과정을 마치고 버킷 생성을 마무리합니다.

ACL 활성화 및 퍼블릭 액세스 차단 설정을 해제한 이유는 이후 acl 설정으로 퍼블릭 읽기권한을 부여하기 위함입니다.

## IAM 사용자 생성

<div>
<img width="1014" alt="5" src="https://github.com/user-attachments/assets/ca0156ee-6422-4539-bc19-abbdfd30ff3e" />   
<br>
<img width="1912" alt="6" src="https://github.com/user-attachments/assets/02c0093e-0507-4975-b8ab-3489026fd138" />   
<br>
<img width="1792" alt="7" src="https://github.com/user-attachments/assets/c9f7d49e-2dbb-475c-a986-59f591788ec5" />   
<br>
<img width="1794" alt="8" src="https://github.com/user-attachments/assets/5baccd11-5bf1-4843-a827-f88567770e30" />   
<br>
<img width="1791" alt="9" src="https://github.com/user-attachments/assets/e1c39cdf-ba0c-437f-8edf-bd3e8fba542f" />   
</div>

위의 과정을 따라 IAM 사용자를 생성합니다.

## IAM 액세스 키 생성

<div>
<img width="1616" alt="10" src="https://github.com/user-attachments/assets/5423707a-690e-4439-a659-28aafca7f183" />   
<br>
<img width="640" alt="11" src="https://github.com/user-attachments/assets/d070beef-51ab-485b-a254-16bd273534ff" />   
<br>
<img width="640" alt="12" src="https://github.com/user-attachments/assets/649f9517-2e6d-405e-b88f-c7ffa0bd6df3" />   
<br>
<img width="640" alt="13" src="https://github.com/user-attachments/assets/fbe20fa6-a9ce-4d8e-b859-f30e63d0766e" />   
</div>

표시된 액세스키, 비밀 액세스 키를 따로 메모하여 저장하거나, `.csv`파일을 다운로드하여 보관합니다. 액세스 키는 Spring properties에 등록하여 AWS 리소스에 접근하는 데에 사용됩니다.   
해당 페이지를 벗어나면 이후 액세스 키를 확인할 수 없어 새로 생성해야하니 잘 저장해두어야 합니다.

# 이미지 업로드 API 구현

AWS 인프라 구축을 마치고 Java 기반 Spring Boot 환경에서 `MultipartFile`방식을 활용하여 AWS S3에 이미지를 업로드하는 방법에 대해 알아보도록 하겠습니다.

## 의존성 설정

```bash
implementation 'software.amazon.awssdk:s3:2.29.50'
```

먼저 Spring Boot에서 AWS S3 리소스 접근을 위해 해당 의존성을 추가합니다.   
필자의 경우 `awssdk:s3 2.x` 버전을 사용하였습니다.   

예제를 다룰 때 `awssdk:s3 1.x`, `awssdk:s3 2.x`, `spring-cloud-starter-aws` 등 여러가지 의존성을 사용하는 모습을 볼 수 있습니다. 각 차이점에 대해 간단하게 알아보겠습니다.   
- `spring-cloud-starter-aws` awssdk:~~ 처럼 AWS의 단일된 리소스를 사용하는게 아닌 AWS 리소스에 통합적으로 접근하기 위해 사용됩니다. (Spring 한정)
- `awssdk:s3 1.x` AWS S3에 단일 리소스 접근에 사용되며, 클라이언트 클래스는 amazonS3Client가 사용됩니다. (Java 8 이전 버전의 코드스타일)
- `awssdk:s3 2.x` AWS S3에 단일 리소스 접근에 사용되며, 클라이언트 클래스는 S3Client가 사용됩니다. (Java 8 이상 버전의 코드스타일)

Spring 환경에서 AWS 리소스에 통합적으로 접근하고자 할때는 `spring-cloud-starter-aws`를, AWS S3 리소스만 다루고자 할때는 `awssdk:s3`의 1.x 또는 2.x를 사용할 수 있습니다.

## application.yml 설정

```yml
spring:
  servlet:
    multipart:
      enabled: true # 멀티파트 업로드 지원여부 (default: true)
      file-size-threshold: 0B # 파일을 디스크에 저장하지 않고 메모리에 저장하는 최소 크기 (default: 0B)
      location: /Users/tao/test # 업로드된 파일이 임시로 저장되는 디스크 위치 (default: WAS가 결정)
      max-file-size: 100MB # 한개 파일의 최대 사이즈 (default: 1MB)
      max-request-size: 100MB # 한개 요청의 최대 사이즈 (default: 10MB)

aws:
  s3:
    access-key: ${AWS_ACCESS_KEY}
    secret-key: ${AWS_SECRET_KEY}
    bucket-name: ${BUCKET_NAME}
  region: ${AWS_REGION}
```

앞서 설명한 multipart에 대한 설정을 하고, IAM 생성 시 발급받은 accessKey와 secretKey를 입력합니다.   
`해당 Key는 외부에 노출되지 않도록 주의`가 필요합니다.

## S3 Config 설정

```java
@Configuration
public class S3Config {

    @Value("${aws.s3.access-key}")
    private String accessKey;
    @Value("${aws.s3.secret-key}")
    private String secretKey;
    @Value("${aws.region}")
    private String region;

    @Bean
    public S3Client s3Client() {
        AwsBasicCredentials awsBasicCredentials = AwsBasicCredentials.create(accessKey, secretKey);
        return S3Client.builder()
                .credentialsProvider(StaticCredentialsProvider.create(awsBasicCredentials))
                .region(Region.of(region))
                .build();
    }
    
}
```

S3Client를 Spring Bean에 등록하고, yml에 설정한 Key와 region 값을 @Value 어노테이션으르 통해 주입합니다.

## Service 업로드 비즈니스 로직 작성

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class S3ImageService {

    private final S3Client s3Client;

    @Value("${aws.s3.bucket-name}")
    private String bucketName;

    // [public 메서드] 외부에서 사용, S3에 저장된 이미지 객체의 public url을 반환
    public List<String> upload(List<MultipartFile> files) {
        // 빈 파일 검증
        if (files.isEmpty()) {
            throw new CustomApplicationException(ErrorCode.EMPTY_FILE);
        }
        // 각 파일을 업로드하고 url을 리스트로 반환
        return files.stream()
                .map(this::uploadImage)
                .toList();
    }

    // [private 메서드] validateFile메서드를 호출하여 유효성 검증 후 uploadImageToS3메서드에 데이터를 반환하여 S3에 파일 업로드, public url을 받아 서비스 로직에 반환
    private String uploadImage(MultipartFile file) {
        validateFile(file.getOriginalFilename()); // 파일 유효성 검증
        return uploadImageToS3(file); // 이미지를 S3에 업로드하고, 저장된 파일의 public url을 서비스 로직에 반환
    }

    // [private 메서드] 파일 유효성 검증
    private void validateFile(String filename) {
        // 파일 존재 유무 검증
        if (filename == null || filename.isEmpty()) {
            throw new CustomApplicationException(ErrorCode.NOT_EXIST_FILE);
        }

        // 확장자 존재 유무 검증
        int lastDotIndex = filename.lastIndexOf(".");
        if (lastDotIndex == -1) {
            throw new CustomApplicationException(ErrorCode.NOT_EXIST_FILE_EXTENSION);
        }

        // 허용되지 않는 확장자 검증
        String extension = filename.substring(lastDotIndex + 1).toLowerCase();
        List<String> allowedExtentionList = Arrays.asList("jpg", "jpeg", "png", "gif");
        if (!allowedExtentionList.contains(extension)) {
            throw new CustomApplicationException(ErrorCode.INVALID_FILE_EXTENSION);
        }
    }

    // [private 메서드] 직접적으로 S3에 업로드
    private String uploadImageToS3(MultipartFile file) {
        // 원본 파일 명
        String originalFilename = file.getOriginalFilename();
        // 확장자 명
        String extension = Objects.requireNonNull(originalFilename).substring(originalFilename.lastIndexOf(".") + 1);
        // 변경된 파일
        String s3FileName = UUID.randomUUID().toString().substring(0, 10) + "_" + originalFilename;

        // 이미지 파일 -> InputStream 변환
        try (InputStream inputStream = file.getInputStream()) {
            // PutObjectRequest 객체 생성
            PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                    .bucket(bucketName) // 버킷 이름
                    .key(s3FileName) // 저장할 파일 이름
                    .acl(ObjectCannedACL.PUBLIC_READ) // 퍼블릭 읽기 권한
                    .contentType("image/" + extension) // 이미지 MIME 타입
                    .contentLength(file.getSize()) // 파일 크기
                    .build();
            // S3에 이미지 업로드
            s3Client.putObject(putObjectRequest, RequestBody.fromInputStream(inputStream, file.getSize()));
        } catch (Exception exception) {
            log.error(exception.getMessage(), exception);
            throw new CustomApplicationException(ErrorCode.IO_EXCEPTION_UPLOAD_FILE);
        }

        // public url 반환
        return s3Client.utilities().getUrl(url -> url.bucket(bucketName).key(s3FileName)).toString();
    }
    
}
```

각 메서드의 역할과 메서드에 대한 세부 설명에 대해 주석으로 설명하였습니다.

## Test용 업로드 Controller 생성

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/s3")
public class S3ImageController {

    private final S3ImageService s3ImageService;

    @PostMapping("/upload")
    public ResponseEntity<List<String>> s3Upload(@RequestPart(value = "image", required = false) List<MultipartFile> multipartFile) {
        List<String> upload = s3ImageService.upload(multipartFile);
        return ResponseEntity.ok(upload);
    }

}
```

`MultipartFile`방식으로 이미지를 업로드하므로 `@RequestPart`를 사용하여 Controller에서 요청 데이터를 받습니다.   

## Postman 요청 테스트

<div>
<img width="1074" alt="18" src="https://github.com/user-attachments/assets/ea256194-c8a1-4753-9a06-316c0b1072ec" />
<br>
<img width="1621" alt="19" src="https://github.com/user-attachments/assets/5c245ae3-eddf-45aa-8436-e2984e039729" />
<br>
<img width="1743" alt="20" src="https://github.com/user-attachments/assets/4f88dc75-a829-4d1e-bf85-36c7a1eb3f6f" />
</div>

Postman을 통해 이미지 업로드 요청 시 S3에 정상적으로 객체가 생성되는 모습을 확인할 수 있습니다.   
추가로 `ACL 설정`을 `퍼블릭 읽기 권한`으로 설정했기 때문에 응답받은 이미지 url을 브라우저에 입력하여 업로드된 이미지를 확인할 수 있습니다.   

<div>
<img width="1075" alt="21" src="https://github.com/user-attachments/assets/82841639-32ca-4fe5-9643-369f4a6c2220" />
<br>
<img width="1077" alt="22" src="https://github.com/user-attachments/assets/3d2a1c51-75dc-4cd0-8422-48518773f253" />
</div>

파일 존재 유무와 확장자에 대한 유효성 검증도 정상적으로 이루어지는 모습을 확인할 수 있습니다.

