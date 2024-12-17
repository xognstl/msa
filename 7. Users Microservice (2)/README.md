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


  <br>

### 4. Users Microservice - 로그인 처리 과정
___

1. 사용자 email, password 입력  

2. AuthenticationFilter 클래스의 attemptAuthentication 메소드가 처리    
- (attemptAuthentication 함수에서 request 정보에 전달했던 데이터 값을 RequestLogin.class로 바꿔서 로그인 처리 요청)    
- UsernamePasswordAuthenticationToken 형태로 바꿔서 사용  

3. UserDetailsService 클래스의 loadUserByUsername 메소드 실행 
- repository 의 findByEmail 을 이용해 해당 데이터 검색 
- Spring security의 User 객체로 변경해서 사용  

4. AuthenticationFilter 클래스의 successfulAuthentication 에서 토큰생성(userId로 emailX)
- email을 이용해 db에서 실제 오브젝트값 가져옴(userDto)
- userDto 객체의 userId를 이용해 JWT 생성
- 생성된 token 값을 response header에 저장
- 클라이언트에게 반환

<br>

### 5. Users Microservice - 로그인 성공 처리
___

- AuthenticationFilter class는 빈으로 등록해서 쓰는게 아니라 스프링 시큐리티에서 getAuthenticationFilter() 를 이용해 인스턴스를 직접 생성해서 사용한다.

- userService.java 에 getUserDetailsByEmail 추가
```java
public interface UserService extends UserDetailsService {
    UserDto getUserDetailsByEmail(String userName);
}

```

- getUserDetailsByEmail 메소드 
```java
@Service
public class UserServiceImpl implements UserService {
    @Override
    public UserDto getUserDetailsByEmail(String email) {
        UserEntity userEntity = userRepository.findByEmail(email);
        if (userEntity == null) {
            throw new UsernameNotFoundException(email);
        }
        UserDto userDto = new ModelMapper().map(userEntity, UserDto.class);
        return userDto;
    }
}
```

- WebSecurity 에서 authenticationFilter 생성 부분 변경
```java
@Configuration
@EnableWebSecurity
public class WebSecurity extends WebSecurityConfigurerAdapter {
    //AuthenticationFilter 객체를 생성하고 반환하는 역할
    private AuthenticationFilter getAuthenticationFilter() throws Exception {
        AuthenticationFilter authenticationFilter = new AuthenticationFilter(authenticationManager(), userService, env);
//        authenticationFilter.setAuthenticationManager(authenticationManager()); // 인증 처리를 하기 위한 메니저

        return authenticationFilter;
    }

}

```

- AuthenticationFilter.java 의 successfulAuthentication 에서 userId 가져오는 로직 추가
```java
@Slf4j
public class AuthenticationFilter extends UsernamePasswordAuthenticationFilter {
    @Override   // 로그인이 성공했을 때 입력 ID, Password가 실제 성공했을 때 어떤 값을 반환 시켜줄건지 처리
    protected void successfulAuthentication(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain chain,
                                            Authentication authResult) throws IOException, ServletException {
        log.debug(((User)authResult.getPrincipal()).getUsername()); // 인증이 완료된 결과값
        String userName = ((User)authResult.getPrincipal()).getUsername();
        UserDto userDetails = userService.getUserDetailsByEmail(userName);  // userId 를 가져올 수 있다.
    }
}
```

<br>

### 6. Users Microservice - JWT 생성
___

- JWT dependency 추가
```yaml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>8.9.1</version>
</dependency>
```

- authentication.filter class 에서 사용할 정보 입력
```yaml
token:
  expiration_time: 86400000
  secret: user_token
```

- userDetails 에서 가져온 userId 를 이용하여 토큰을 생성하는 로직 추가
```java
@Slf4j
public class AuthenticationFilter extends UsernamePasswordAuthenticationFilter {
    @Override
    protected void successfulAuthentication(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain chain,
                                            Authentication authResult) throws IOException, ServletException {
        log.debug(((User)authResult.getPrincipal()).getUsername()); // 인증이 완료된 결과값
        String userName = ((User)authResult.getPrincipal()).getUsername();
        UserDto userDetails = userService.getUserDetailsByEmail(userName);  // userId 를 가져올 수 있다.

        String token = Jwts.builder()
                .setSubject(userDetails.getUserId())    //토큰 생성
                .setExpiration(new Date(System.currentTimeMillis()
                        + Long.parseLong(env.getProperty("token.expiration_time"))))
                .signWith(SignatureAlgorithm.HS512, env.getProperty("token.secret")) // 암호화
                .compact();

        response.addHeader("token", token);
        response.addHeader("userId", userDetails.getUserId());
    }
}
```
- POST 127.0.0.1:8000/user-service/login => header 에서 token 확인 가능 

<br>

### 7. Users Microservice - JWT 처리 과정
___
- 세션, 쿠키로 인증 => 모바일에서 유효하게 사용X
- 토큰을 이용해 사용자가 정상적으로 로그인이 된것을 서버측에 알려주면서 인증처리가 유효하게 만든다.
- 클라이언트 독립적인 서비스, 캐시서버 인증처리 가능, 지속적인 토큰 저장
- 
- 토큰값 jwt.io 에서 decode 하면 아래와 같이 나온다.
```json
{
  "sub": "40bebf71-a06c-4ca3-8f1d-997f3916da13",
  "exp": 1734488087
}
```

- API Gateway Service 에 Spring Security, JWT Token 사용 추가

- dependency 추가
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```
- api Gateway service 에 AuthorizationHeaderFilter.java 생성
  
- yaml 에 AuthorizationHeaderFilter 추가
```yaml
- id: user-service
  uri: lb://USER-SERVICE
  predicates:
    - Path=/user-service/**
    - Method=GET
  filters:
    - RemoveRequestHeader=Cookie #초기화
    - RewritePath=/user-service/(?<segment>.*), /$\{segment}
    - AuthorizationHeaderFilter

token:
  secret: user_token
```
  
- API 를 호출할 때 사용자가 헤더에 토큰을 전달해주는 작업
- apply 함수 로그인 -> 토큰 받음 -> user(토큰 정보가지고 요청) -> header(include token)
```java
@Component
@Slf4j
public class AuthorizationHeaderFilter extends AbstractGatewayFilterFactory<AuthorizationHeaderFilter.Config> {

    Environment env;

    public AuthorizationHeaderFilter(Environment env) {
        this.env = env;
    }
    @Override
    public GatewayFilter apply(Config config) {
        return ((exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            if(!request.getHeaders().containsKey(HttpHeaders.AUTHORIZATION)){   // AUTHORIZATION 관련된 값 존재 유무
                return onError(exchange, "no Authorization header", HttpStatus.UNAUTHORIZED);
            }

            String authorizationHeader = request.getHeaders().get(HttpHeaders.AUTHORIZATION).get(0);    // 토큰 값 가져오기
            String jwt = authorizationHeader.replace("Bearer", "");
            
            if (!isJwtValid(jwt)) { // Jwt token valid
                return onError(exchange, "JWT token is not valid", HttpStatus.UNAUTHORIZED);
            }
            
            return chain.filter(exchange);
        });
    }

    // jwt decode 했을때 Subject 를 토큰으로 부터 추출한 후 값이 정상적인 계정 값인지 판단
    private boolean isJwtValid(String jwt) {
        boolean returnValue = true;

        String subject = null;
        try {
            subject = Jwts.parser().setSigningKey(env.getProperty("token.secret"))
                    .parseClaimsJws(jwt).getBody()
                    .getSubject();  // 토큰을 문자형 데이터 값으로 파싱하기 위해 parseClaimsJws로 파싱 , subject 값만 추출
        } catch (Exception exception) {
            returnValue = false;
        }

        if (subject == null || subject.isEmpty()) {
            returnValue = false;
        }

        return returnValue;
    }

    // Mono(단일), Flux(다중)
    private Mono<Void> onError(ServerWebExchange exchange, String err, HttpStatus httpStatus) {
        ServerHttpResponse response = exchange.getResponse();   // response 객체 받아옴
        response.setStatusCode(httpStatus);

        log.error(err);

        return response.setComplete();
    }
    

    public static class Config {

    }
}
```
