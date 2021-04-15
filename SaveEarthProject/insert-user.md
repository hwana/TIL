# 유저 정보 데이터베이스에 저장하기

> 구글 oAuth 개발을 마치고 4월 10일 수업에 참석했다. 수업은 구글 oAuth에 관한 내용으로 진행이 되었고, 프론트에서 구글 oAuth 작업을 훨씬 간결한 코드로 구현 가능하다는 것을 알게되었다.
> 이미 해놓은 작업이 조금 아쉬웠지만 굳이 백엔드 서버를 더 거쳐서 로그인이 진행 될 필요가 없다고 판단이 되었기 때문에 구글 oAuth 기능은 프론트에서 구현하기로 결정했다.
> 나는 프론트에서 넘겨주는 데이터를 데이터베이스에 저장해야하는 기능을 구현해야한다.

## User.java
```java
@Getter
@NoArgsConstructor
@Entity
@Table(name = "users")
public class User {

    @Id
    @Column(name = "USER_ID")
    private String id;

    @NotNull
    private String email;

    @NotNull
    private String name;

    @Column
    private String imgUrl;

    @OneToMany(mappedBy = "userLike")
    private List<Like> likeList = new ArrayList<Like>();

    @OneToMany(mappedBy = "userComment")
    private List<Comment> commentList = new ArrayList<Comment>();

    @Builder
    public User(String id, String email, String name, String imgUrl) {
        this.id = id;
        this.email = email;
        this.name = name;
        this.imgUrl = imgUrl;
    }
}
```
- 기본키 매핑 전략을 직접 할당으로 변경했다.

## UserSaveRequestDto.java
```java
@Getter
@Setter
@NoArgsConstructor
public class UserSaveRequestDto {
    private String id;
    private String email;
    private String name;
    private String imgUrl;

    public User toEntity(){
        return User.builder()
                .id(id)
                .email(email)
                .name(name)
                .imgUrl(imgUrl)
                .build();
    }
}
```

## UserRepository.java
```java
public interface UserRepository extends JpaRepository<User, Long> {

}
```

## UserService.java
```java
@Service
@Transactional
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public void saveUser(UserSaveRequestDto userSaveRequestDto){
        userSaveRequestDto.setId("구글 유니크 키값");
        userRepository.save(userSaveRequestDto.toEntity());
    }
}
```
- 기본키 매핑전략을 직접 할당으로 변경했기 때문에 save 하기 전에 애플리케이션에서 기본키를 직접 할당해 주어야 한다.
- 현재는 임의의 값을 설정해 놓았고, 프론트에서 구글 유니크 값이 넘어 오면 그 값을 설정하려고 한다.

## UserController.java
```java
@RestController
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;

    @PostMapping("/api/user")
    public void saveUser(UserSaveRequestDto userSaveRequestDto){
        userService.saveUser(userSaveRequestDto);
    }
}
```
