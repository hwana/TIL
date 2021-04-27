# 유저 정보 조회하기
> 우리 서비스의 기능 중 유저 정보를 필요로 하는 기능이 없기 때문에 유저 정보 조회하는 API를 구현하지 않았다. 
> 하지만 중복회원을 구분해 내기 위해서 유저 정보를 조회하는 API가 필요할 것 같다는 멘토님에 피드백으로 구현을 하게 되었다.
> 

## UserController.java
```java
@RestController
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;

    @GetMapping("/api/user/{userId}")
    public String getUser(@PathVariable("userId") String userId){
        return userService.selectUser(userId);
    }
}
```
## UserService.java
```java
@Service
@Transactional
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public String selectUser(String userId){
        Optional<User> user = userRepository.findById(userId);
        if(user.isPresent()){
            return user.get().getId();
        }else{
            return null;
        }
    }
}
```
- 유저 아이디로 조회하여 아이디가 있다면 아이디값을 반환하고, 아이디가 없다면 null 값을 반환한다.
