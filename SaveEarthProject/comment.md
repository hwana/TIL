# 댓글 등록, 수정, 삭제, 조회

> 게시글에 댓글이 달릴 수 있으며, 댓글이 10개 이상일때는 페이징이 된다. 댓글을 수정할 때는 제목만 수정이 되며 삭제가 가능하다.

## 댓글 삭제
### CommentController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class CommentController {
    private final CommentService commentService;

    @DeleteMapping("/comment/{commentId}")
    public void deleteComment(@PathVariable(value = "commentId") Long id){
        commentService.deleteComment(id);
    }
}
````
### CommentService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommentService {

    private final CommentRepository commentRepository;
    
    public void deleteComment(Long id){
        commentRepository.deleteById(id);
    }
}
```
## 댓글 수정
### Comment.java
```java
@Getter
@NoArgsConstructor
@Entity
@Table(name = "comments")
public class Comment extends BaseTimeEntity{
    .
    .
    .

    public void changeContents(String contents){
        this.contents = contents;
    }
}
```
- JPA에서 제공하는 변경감지 기능을 사용하기 위해서 changeContents 메소드를 생성하였다.

### CommentUpdateRequestDto.java
```java
@Getter
@NoArgsConstructor
public class CommentUpdateRequestDto {
    private String contents;
}
```
### CommentController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class CommentController {

    private final CommentService commentService;

    @PatchMapping("/comment/{commentId}")
    public void updateComment(@PathVariable(value = "commentId") Long id, @RequestBody CommentUpdateRequestDto commentUpdateRequestDto){
        commentService.updateComment(id, commentUpdateRequestDto.getContents());
    }
}
```
### CommentService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommentService {

    private final CommentRepository commentRepository;

    public void updateComment(Long id, String contents){
        Comment comment = commentRepository.getOne(id);
        comment.changeContents(contents);
    }
}
```
- getOne(id) 메소드를 사용하여 1차 캐시에 저장되어 있던 객체를 조회하여, 해당 객체를 변경감지 기능으로 수정을 했다.

## 댓글 조회
> 댓글 조회 시 필요한 데이터는
> 1. 사용자 이름
> 2. 사용자 이미지 url
> 3. 댓글 내용
> 4. 댓글 작성 시간
> 이다.<br>
>
> 에코 플레이스의 아이디를 받아서 사용자에 대한 정보도 조회해야 하기 때문에 Comment 엔티티를 기준으로 User 엔티티를 조인해줬다.<br> 

### build.gradle
```java
plugins {
	id 'org.springframework.boot' version '2.4.4'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
}

apply plugin: "io.spring.dependency-management"

group = 'com.save.earth'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'junit:junit:4.13.1'

	implementation("com.querydsl:querydsl-core")
	implementation("com.querydsl:querydsl-jpa")

	annotationProcessor("com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa") // querydsl JPAAnnotationProcessor 사용 지정
	annotationProcessor("jakarta.persistence:jakarta.persistence-api") // java.lang.NoClassDefFoundError(javax.annotation.Entity) 발생 대응
	annotationProcessor("jakarta.annotation:jakarta.annotation-api") // java.lang.NoClassDefFoundError (javax.annotation.Generated) 발생 대응

    compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	runtimeOnly 'mysql:mysql-connector-java'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

clean {
	delete file('src/main/generated') 
}

task cleanGeneatedDir(type: Delete) { 
	delete file('src/main/generated')
}

test {
	useJUnitPlatform()
}

```
- 현재 gradle 6.8.3 버전을 사용하고 있기 때문에 그에 맞는 querydsl 적용 방법이 필요했다. 그래서 [이 블로그](http://honeymon.io/tech/2020/07/09/gradle-annotation-processor-with-querydsl.html)를 많이 참고하고 build.gradle을 수정했다.

### CommentResponseDto.java
```java
@Getter
@NoArgsConstructor
public class CommentResponseDto {

    private String contents;
    private String userName;
    private String userImgUrl;
    private LocalDateTime createDateTime;

    public CommentResponseDto(Comment comment){
        this.contents = comment.getContents();
        this.userName = comment.getUserComment().getName();
        this.userImgUrl = comment.getUserComment().getImgUrl();
        this.createDateTime = comment.getCreateDateTime();
    }
}
```

### CommentRepositoryCustom.java
```java
@Repository
@RequiredArgsConstructor
public class CommentRepositoryCustom {
    private final JPAQueryFactory jpaQueryFactory;

    public List<Comment> findComment(Long placeId){
        QPlace qPlace = QPlace.place;
        QComment qComment = QComment.comment;
        QUser qUser = QUser.user;

        return jpaQueryFactory.select(qComment)
            .from(qComment)
            .innerJoin(qComment.userComment, qUser)
            .fetchJoin()
            .where(qComment.placeComment.id.eq(placeId))
            .fetch();
    }
}
```
- querydsl을 사용하기 위해서 별도의 custom repository를 생성했다. 
- 컬렉션을 조인하면 발생하는 N+1문제를 해결하기 위해서 fetch join을 사용했다.
> 처음 querydsl을 작성할 때는 place에 댓글 리스트만 가져오면 된다고 생각해서 place 엔티티 기준으로 데이터를 조회했었다. 그랬더니 사용자 정보를 가져올 수가 없어서 다시 comment 엔티티
> 기준으로 작성했더니 placeId로 댓글리스트와 유저정보 모두 다 가져올 수 있었다.

### CommentService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommentService {

    private final CommentRepository commentRepository;
    private final CommentRepositoryCustom commentRepositoryCustom;

    public List<CommentResponseDto> selectComment(Long placeId){
        return commentRepositoryCustom.findComment(placeId).stream().map(CommentResponseDto::new).collect(Collectors.toList());
    }
}
```

### CommentController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class CommentController {

    private final CommentService commentService;

    @GetMapping("/comment/{placeId}")
    public List<CommentResponseDto> commentList(@PathVariable(value = "placeId") Long placeId){
        return commentService.selectComment(placeId);
    }
}
```
## 댓글 등록
### CommentSaveRequestDto.java
```java
@Getter
@NoArgsConstructor
public class CommentSaveRequestDto {

    private String contents;
    private Long placeId;
    private String userId;

}
```
### CommentService.java
```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommentService {

    private final CommentRepository commentRepository;
    private final PlaceRepository placeRepository;
    private final UserRepository userRepository;

    public void saveComment(CommentSaveRequestDto commentSaveRequestDto) {

        Place place = placeRepository.getOne(commentSaveRequestDto.getPlaceId());
        User user = userRepository.getOne(commentSaveRequestDto.getUserId());
        String contents = commentSaveRequestDto.getContents();

        commentRepository.save(Comment.builder().placeComment(place).userComment(user).contents(contents).build());
    }
}
```
### CommentController.java
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class CommentController {

    private final CommentService commentService;

    @PostMapping("/comment")
    public void saveComment(@RequestBody CommentSaveRequestDto commentSaveRequestDto){
        commentService.saveComment(commentSaveRequestDto);
    }
}
```
