# oAuth 2.0을 이용한 구글로그인 구현하기

### 1. [구글 클라우드 플랫폼](https://console.cloud.google.com/getting-started)에서 사용자 인증정보 생성 후 클라이언트 ID, 비밀번호 확인
### 2. src/main/resources에 application-oauth.properties 파일 생성
```
spring.security.oauth2.client.registration.google.client-id=구글클라이언트ID
spring.security.oauth2.client.registration.google.client-secret=구글클라이언트시크릿
spring.security.oauth2.client.registration.google.scope=profile,email
```
- scope를 별도로 등록하지 않으면 기본값은 openid, profile, emial이다.
- 여기서 profile, email을 scope로 강제로 등록한 이유는 openid라는 scope가 있으면 Open Id Provider로 인식하기 때문이다.
- 이렇게 되면 OpenId Provider인 서비스(구글)와 그렇지 않은 서비스(네이버 등)로 나눠서 각각 OAuth2Service를 만들어야 한다.
- 하나의 OAuth2Service로 사용하기 위해 일부러 openid scope를 빼고 등록한다.

### 3. `.gitignore`에 application-oauth.properties 코드 추가
- 구글 로그인을 위한 클라이언트 ID/비밀번호는 보안상 중요한 정보들인데 git hub에 연동해서 사용하다보니 노출될 가능성이 높다.
- 그래서 .gitignore 파일에 등록하여 application-auth.properties 파일이 업로드 되는것을 방지해야 한다.

### 4. build.gradle에 스프링 시큐리티 의존성 추가<br>
`implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'`

### 5. SecurityConfig 클래스 생성
config.auth 패키지를 생성 - 시큐리티 관련 클래스는 모두 이곳에 담으면 됨
```java
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private final CustomOAuth2UserService customOAuth2UserService;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers("/", "/css/**", "/images/**", "/js/**").permitAll()
                .anyRequest().authenticated()
                .and()
                .logout()
                .logoutSuccessUrl("/")
                .and()
                .oauth2Login()
                .userInfoEndpoint()
                .userService(customOAuth2UserService);
    }
}
```
- @EnableWebSecurity : Spring Security 설정들을 활성화시켜줌
- .authorizeRequests() : URL별 권환 관리를 설정하는 옵션의 시작점
- .antMatchers() : 권한 관리 대상을 지정하는 옵션으로 URL, HTTP 메소드별로 관리가 가능함
- .permitAll() : 지정된 URL들에게 전체 열람 권한을 주는 옵션
- .anyRequest() : 설정된 값들 이외의 나머지 URL
- .authenticated() : 나머지 URL들은 모두 인증된 사용자(로그인한 사용자)들에게만 허용하는 옵션
- .logout().logoutSuccessUrl("/") : 로그아웃 기능에 대한 설정 진입점, 로그아웃 성공시 / 주소로 이동
- .oauth2Login() : OAuth2 로그인 기능에 대한 설정의 진입점
- .userInfoEndpoint() : OAuth2 로그인 성공 이후 사용자 정보를 가져올 때의 설정을 담당하는 옵션
- .userService() : 소셜 로그인 성공 시 후속 조치를 진행할 UserService 인터페이스의 구현체를 등록함

### 6. CustomOAuth2UserService 클래스 생성
- 해당 클래스는 구글 로그인 후 가져온 사용자의 정보(email, picture, name 등)을 기반으로 가입 및 정보수정, 세션 저장등의 기능을 지원함
```java
@RequiredArgsConstructor
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {
    private final UserRepository userRepository;
    private final HttpSession httpSession;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2UserService delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);

        String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails()
                .getUserInfoEndpoint().getUserNameAttributeName();

        OAuthAttributes attributes = OAuthAttributes.of(userNameAttributeName, oAuth2User.getAttributes());

        User user = saveOrUpdate(attributes);
        httpSession.setAttribute("user", new SessionUser(user));

        return new DefaultOAuth2User(
                Collections.singleton(new SimpleGrantedAuthority(user.getRoleKey())),
                attributes.getAttributes(),
                attributes.getNameAttributeKey());
    }


    private User saveOrUpdate(OAuthAttributes attributes) {
        User user = userRepository.findByEmail(attributes.getEmail())
                .map(entity -> entity.update(attributes.getName(), attributes.getImgUrl()))
                .orElse(attributes.toEntity());

        return userRepository.save(user);
    }
}
```
- userNameAttributeName : OAuth2 로그인 진행시 키가되는 필드값, 구글은 기본적으로 코드(sub)를 지원, 네이버와 카카오는 지원하지 않음
- OAuthAttributes : OAuth2UserService를 통해 가져온 OAuth2User의 attribute를 담을 클래스
- SessionUser : 세션에 사용자 정보를 저장하기 위한 Dto 클래스
- 구글 사용자 정보가 업데이트 되었을 때를 대비하여 saveOrUpdate 기능 구현

### 7. OAuthAttributes 클래스 생성
- config>auth>dto 패키지 생성
```java
@Getter
public class OAuthAttributes {
    private Map<String, Object> attributes;
    private String nameAttributeKey;
    private String name;
    private String email;
    private String picture;

    @Builder
    public OAuthAttributes(Map<String, Object> attributes, String nameAttributeKey, String name, String email, String picture) {
        this.attributes = attributes;
        this.nameAttributeKey = nameAttributeKey;
        this.name = name;
        this.email = email;
        this.picture = picture;
    }

    public static OAuthAttributes of(String registrationId, String userNameAttributeName, Map<String, Object> attributes) {
           return ofGoogle(userNameAttributeName, attributes);
    }

    private static OAuthAttributes ofGoogle(String userNameAttributeName, Map<String, Object> attributes) {
        return OAuthAttributes.builder()
                .name((String) attributes.get("name"))
                .email((String) attributes.get("email"))
                .picture((String) attributes.get("picture"))
                .attributes(attributes)
                .nameAttributeKey(userNameAttributeName)
                .build();
    }

    public User toEntity() {
        return User.builder()
                .name(name)
                .email(email)
                .picture(picture)
                .build();
    }
}
```
- of() : OAuth2User에서 반환하는 사용자 정보는 Map이기 때문에 값 하나하나를 변환해야만 함
- toEntity() : User 엔티티를 생성, OAuthAttribute에서 엔티티를 생성하는 시점은 처음 가입할 때

### 8. SessionUser 클래스 생성
- config>auth>dto 패키지에 생성
```java
@Getter
public class SessionUser implements Serializable {
    private String name;
    private String email;
    private String picture;

    public SessionUser(User user) {
        this.name = user.getName();
        this.email = user.getEmail();
        this.picture = user.getPicture();
    }
}
```
- SessionUser에는 인증된 사용자 정보만 필요하기 때문에 name, email, picture만 필드로 선언함

### 9. 어노테이션 기반으로 코드 개선하기
>IndexController 외 다른 컨트롤러와 메소드에서 세션값이 필요할때마다 SessionUser user = (SessionUser) httpSession.getAttribute("user");를 입력하여 직접 세션에서 가져와야 한다. 
>그렇게 되면 같은 코드가 계속해서 반복되고, 이 코드가 수정되야할 경우가 생긴다면 동일한 코드를 다 찾아서 수정해줘야 한다. 
>이런 불필요한 행동을 줄이기 위해서 메소드 인자로 세션값을 바로 받을 수 있도록 변경 할 것이다.

### 9-1. @LoginUser 어노테이션 생성
- config>auth 패키지에 생성
```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoginUser {
}
```
- @Target(ElementType.PARAMETER) : 어노테이션이 생성될 수 있는 위치를 지정함, PARAMETER로 지정했으니 메소드의 파라미터로 선언된 객체에서만 사용이 가능하다.
- @interface : 이 파일을 어노테이션 클래스로 지정함, LoginUser라는 이름을 가진 어노테이션이 생성되었다고 보면 됨

### 9-2. LoginUserArgumentResolver 생성
- @LoginUser와 같은 위치에 생성
- HandlerMethodArgumentResolver를 구현한 클래스
- HandlerMethodArgumentResolver의 기능 : 컨트롤러 메소드에서 특정 조건에 맞는 파라미터가 있을 때 원하는 값을 바인딩해주는 인터페이스
```java
@RequiredArgsConstructor
@Component
public class LoginUserArgumentResolver implements HandlerMethodArgumentResolver {
    private final HttpSession httpSession;

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        boolean isLoginUserAnnotation = parameter.getParameterAnnotation(LoginUser.class) != null;
        boolean isUserClass = SessionUser.class.equals(parameter.getParameterType());
        return isLoginUserAnnotation && isUserClass;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        return httpSession.getAttribute("user");
    }
}
```
- supportsParameter() : 파라미터에 @LoginUser 어노테이션이 붙어있고, 파라미터 클래스 타입이 SessionUser.class 인 경우 true를 반환함
- resolveArgument() : 파라미터에 전달할 객체를 생성함, 여기에서는 세션 객체를 가져옴

### 9-3. WebConfig 클래스 생성
- LoginUserArgumentResolver가 스프링에서 인식될 수 있도록 WebMvcConfigurer에 추가해줌
- HandlerMethodArgumentResolver는 항상 WebMvcConfigurer의 addArgumentResolvers()를 통해 추가해야함
```java
@RequiredArgsConstructor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    private final LoginUserArgumentResolver loginUserArgumentResolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(loginUserArgumentResolver);
    }
}
```
