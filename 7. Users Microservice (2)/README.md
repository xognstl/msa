## 섹션 7. Users Microservice ➁

### 1. Users Microservice - AuthenticationFilter 추가
___

- AuthenticationFilter : Spring Security 를 이용한 로그인 요청 발생 시 작업을 처리해주는 클래스
- RequestLogin.java - 사용자가 전달했던 데이터 값을 저장하기 위한 클래스
```java
@Data
public class RequestLogin {
    @NotNull(message = "Email cannot be null")
    @Size(min = 2, message = "Email not be less than 2 characters")
    @Email
    private String email;

    @NotNull(message = "Password cannot be null")
    @Size(min = 8, message = "Password not be equals or grater than 8 characters")
    private String password;
}
```

- AuthenticationFilter.java 추가
- attemptAuthentication : 요청 정보를 보냈을때 그것을 처리 시켜줄 수 있는 메소드,   
로그인 인증 과정에서 사용, 사용자가 제공한 인증 정보를 기반으로 인증을 시도하고, 성공하거나 실패한 결과를 반환합니다.
- 사용자가 입력했던 email,id값을 Spring Security 에서 사용할 수 있는 형태의 값으로  
변환하기 위해 UsernamePasswordAuthenticationToken 이라는 형태로 바꿔줘야 한다. 

```java
public class AuthenticationFilter extends UsernamePasswordAuthenticationFilter {
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request,
                                                HttpServletResponse response) throws AuthenticationException {

        try {
            RequestLogin creds = new ObjectMapper().readValue(request.getInputStream(), RequestLogin.class);
            // post 형태로 로그인 정보가 전달 되기 때문에 requestParameter로 받을수 없으므로 InputStream 사용

            return getAuthenticationManager().authenticate(
                    new UsernamePasswordAuthenticationToken(creds.getEmail(), creds.getPassword(), new ArrayList<>()));
            // token을 Authentication 매니저에 인증작업을 요청
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```
- WebSecurity.java 에서 filter, ip추가 및 getAuthenticationFilter함수 추가(인증처리)
```java
@Configuration
@EnableWebSecurity // web security 용도로 사용할 것이다.
public class WebSecurity extends WebSecurityConfigurerAdapter {
    @Override   // 권한
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
        http.authorizeRequests().antMatchers("/**")
                .hasIpAddress("127.0.0.1")
                .and()
                .addFilter(getAuthenticationFilter());
        
        http.headers().frameOptions().disable();    
    }

    private AuthenticationFilter getAuthenticationFilter() throws Exception {
        AuthenticationFilter authenticationFilter = new AuthenticationFilter();
        authenticationFilter.setAuthenticationManager(authenticationManager()); 
        // 인증 처리를 하기 위한 메니저
        return authenticationFilter;
    }
}
```

<br>

### 2. Users Microservice - loadUserByUsername() 구현
___

- userDetailsService : 사용자가 전달한 username, password로 로그인 처리
- WebSecurity.java 의 configure 함수 추가
```java
@Configuration
@EnableWebSecurity // web security 용도로 사용할 것이다.
public class WebSecurity extends WebSecurityConfigurerAdapter {

    private UserService userService;
    private BCryptPasswordEncoder bCryptPasswordEncoder;
    private Environment env;

    @Autowired
    public WebSecurity(UserService userService, BCryptPasswordEncoder bCryptPasswordEncoder, Environment env) {
        this.userService = userService;
        this.bCryptPasswordEncoder = bCryptPasswordEncoder;
        this.env = env;
    }
    //select pwd from users where email = ?
    // db_pwd(encrypted) == input_pwd(encrypted)
    @Override   //인증
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService).passwordEncoder(bCryptPasswordEncoder);
    }
}
```
- UserService.java에 UserDetailsService 상속
- UserServiceImpl에 상속받은 함수 오버라이드
```java
public class UserServiceImpl implements UserService {
    @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            UserEntity userEntity = userRepository.findByEmail(username);
            if (userEntity == null) {   // 아이디가 없을 떄
                throw new UsernameNotFoundException(username);
            }
            return new User(userEntity.getEmail(), userEntity.getEncryptedPwd(), true, true, true, true, new ArrayList<>());//arrayList(권한)
    
    }
}
```

- UserRepository 에 findByEmail 추가
```java
public interface UserRepository extends CrudRepository<UserEntity, Long> {
    UserEntity findByEmail(String username);
}
```

<br>

### 3. Users Microservice - Routes 정보 변경
___

- api gateway yaml 
- login, 회원가입, 나머지 구분
```yaml
- id: user-service
  uri: lb://USER-SERVICE
  predicates:
    - Path=/user-service/login
    - Method=POST
  filters:
    - RemoveRequestHeader=Cookie #초기화
    - RewritePath=/user-service/(?<segment>.*), /$\{segment}
- id: user-service
  uri: lb://USER-SERVICE
  predicates:
    - Path=/user-service/users
    - Method=POST
  filters:
    - RemoveRequestHeader=Cookie #초기화
    - RewritePath=/user-service/(?<segment>.*), /$\{segment}
- id: user-service
  uri: lb://USER-SERVICE
  predicates:
    - Path=/user-service/**
    - Method=GET
  filters:
    - RemoveRequestHeader=Cookie #초기화
    - RewritePath=/user-service/(?<segment>.*), /$\{segment}
```
- POST 127.0.0.1:8000/user-service/login => 200 정상 , 비밀번호 틀리면 401 Unauthorized 에러
