# AWS S3 이미지 업로드 하기
> 프론트와 API연동을 하기 위해서 이제 서버를 배포해야 할 시기가 왔다. 더이상 로컬 서버에 업로드 하지않고 AWS에서 제공하는 S3에 이미지를 업로드 하려고 한다. 
> 처음 해보는 작업이기 때문에 이 [블로그](https://victorydntmd.tistory.com/334)를 많이 참고했다. S3 버킷을 생성하고 사용자를 생성하여 키를 발급받고 코드작성을 시작했다.

## 1.로컬에서 이미지 업로드 하기
### build.gradle
`implementation group: 'org.springframework.cloud', name: 'spring-cloud-starter-aws', version: '2.2.6.RELEASE'` 추가 <br>
- 처음에는 참고한 블로그에 나와있는대로 `spring-cloud-aws` 의존성을 추가하여 진행하였는데 무슨 이유 때문인지는 모르겠지만 의존성이 제대로 추가되지 않았다.
- 그래서 메이븐 리파지토리에서 의존성을 다시 검색해서 `spring-cloud-starter-aws`를 추가했다.

### application.properties
```
cloud.aws.credentials.secret-key=비밀키
cloud.aws.credentials.access-key=액세스키
cloud.aws.region.static=ap-northeast-2
cloud.aws.stack.auto=false
cloud.aws.s3.bucket=버킷이름
```
- 이 파일은 AWS KEY에 대한 정보가 작성되어 있기 때문에 절대로 Github에 업로드 되면 안된다.
- 그래서 .gitignore에 포함시켜 주었다.

### PlaceService.java
```java
@Service
@Transactional
@RequiredArgsConstructor
public class PlaceService {
    private final PlaceRepository placeRepository;
    private final S3Service s3Service;

    public void savePlace(PlaceSaveRequestDto placeSaveRequestDto, MultipartFile file){
        try {

            String imgPath = s3Service.upload(file);
            placeSaveRequestDto.setImgUrl(imgPath);

            placeRepository.save(placeSaveRequestDto.toEntity());

        }catch(Exception e){
            e.printStackTrace();
        }
    }
}
```

### S3Service.java
```java
@Service
@NoArgsConstructor
public class S3Service {
    private AmazonS3 s3Client;

    @Value("${cloud.aws.credentials.accessKey}")
    private String accessKey;

    @Value("${cloud.aws.credentials.secretKey}")
    private String secretKey;

    @Value("${cloud.aws.s3.bucket}")
    private String bucket;

    @Value("${cloud.aws.region.static}")
    private String region;

    @PostConstruct
    public void setS3Client() {
        AWSCredentials credentials = new BasicAWSCredentials(this.accessKey, this.secretKey);

        s3Client = AmazonS3ClientBuilder.standard()
                .withCredentials(new AWSStaticCredentialsProvider(credentials))
                .withRegion(this.region)
                .build();
    }

    public String upload(MultipartFile file) throws IOException {
        String fileName = file.getOriginalFilename();

        s3Client.putObject(new PutObjectRequest(bucket, fileName, file.getInputStream(), null)
                .withCannedAcl(CannedAccessControlList.PublicRead));
        return s3Client.getUrl(bucket, fileName).toString();
    }
}
```

## 2. 배포해서 업로드하기
> EC2에서 S3를 사용하기 위해서 역할을 추가해서 설정해줬다. 이 [블로그](https://jojoldu.tistory.com/300)를 많이 참고했다.
> 설정된 역할에 따라 EC2는 액세스 키와 비밀키를 할당받아서 사용한다. 그래서 `application.properties`와 `S3Service.java`에서 키와 관련된 코드는 삭제해줬다.

### 수정된 application.properties
```java
cloud.aws.credentials.instance-profile=true
cloud.aws.region.static=ap-northeast-2
cloud.aws.stack.auto=false
cloud.aws.s3.bucket=버킷이름
```

### 수정된 S3Service.java
```java
@Service
@NoArgsConstructor
public class S3Service {
    private AmazonS3 s3Client;

    @Value("${cloud.aws.s3.bucket}")
    private String bucket;

    @Value("${cloud.aws.region.static}")
    private String region;

    @PostConstruct
    public void setS3Client() {

        s3Client = AmazonS3ClientBuilder.standard()
                .withRegion(this.region)
                .build();
    }

    public String upload(MultipartFile file) throws IOException {
        String fileName = file.getOriginalFilename();

        s3Client.putObject(new PutObjectRequest(bucket, fileName, file.getInputStream(), null)
                .withCannedAcl(CannedAccessControlList.PublicRead));
        return s3Client.getUrl(bucket, fileName).toString();
    }
}
```

### 테스트 결과 확인
![image](https://user-images.githubusercontent.com/37647995/116078654-d82aee80-a6d1-11eb-8e6a-06295009dec7.png)
![image](https://user-images.githubusercontent.com/37647995/116078715-ea0c9180-a6d1-11eb-8292-bb1733437fdb.png)<br>
사진이 업로드 된 것을 확인할 수 있다.

