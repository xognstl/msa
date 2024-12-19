## 섹션 5. Users Microservice (1)

### 1. Users Microservice 개요
___
- 신규 회원 등록
- 회원 로그인
- 상세 정보 확인
- 회원 정보 수정/삭제
- 상품 주문
- 주문 내역 확인

<br>

### 2. Users Microservice - 프로젝트 생성
___
- dependencies : devTools, Lombok, Web, Eureka Discovery Client
- Spring boot 2.4.x
- H2 Database

- @EnableDiscoveryClient : 유레카 서버에 등록
```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
}
```

- UserController 생성
```java
@RestController
@RequestMapping("/")
public class UserController {
    @GetMapping("/heath_check")
    public String status() {
        return "It's Working in User Service";
    }
}
```
- Eureka 서버 기동할때 인텔리제이 대신 cmd로 기동
- msa_project\discoveryservice> mvn spring-boot:run
- 기존 유레카 서버 등록, instance-id , 랜덤 포트는 application.yml 에 구현되어있다.
- user-service 실행 후 http://127.0.0.1:61441/heath_check => 정상 작동 확인

```yaml
greeting:
  message: Welcome to the simple E-commerce
```
- yaml 파일에 메시지 추가, 메시지 출력 방법은 1. Environment 객체 , 2. @Value 사용

1. Environment 객체 사용
```java
    private Environment env;
    @Autowired
    public UserController(Environment env) {
        this.env = env;
    }
    @GetMapping("/welcome")
    public String welcome() {
        return env.getProperty("greeting.message");
    }
```
- http://127.0.0.1:61552/welcome => greeting.message 정상 출력 확인

2. @Value 사용
- vo.Greeting.java 생성
```java
@Component
@Data
public class Greeting {
    @Value("${greeting.message}")
    private String message;    
}
```

```java
    @Autowired
    private Greeting greeting;
    
@GetMapping("/welcome")
    public String welcome() {
        return greeting.getMessage();
    }
```
- http://127.0.0.1:61657/welcome => greeting.message 정상 출력 확인

<br>

### 3. Users Microservice - H2 데이터베이스 연동
___
- h2 database dependency 추가
```yaml
<!-- https://mvnrepository.com/artifact/com.h2database/h2 -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>1.3.176</version>
    <scope>runtime</scope>
</dependency>
```

```yaml
  h2:
    console:
      enabled: true
      settings:
        web-allow-others: true # 외부에서 접속
      path: /h2-console
```
- http://127.0.0.1:62951/h2-console/ 하면 h2 콘솔에 접속할 수 있다.

<br>

### 4. Users Microservice - 사용자 추가
___

#### 회원 가입

- modelmapper : dto 객체 => userEntity 객체로 바꿔줄 수 있다.
- 데이터 흐름 : requestUser.java -> UserDto.java -> userEntity

- RequestUser.java
```java
@Data
public class RequestUser {
    @NotNull(message = "Email cannot be null")
    @Size(min = 2, message = "Email not be less than two characters")
    @Email
    private String email;

    @NotNull(message = "Name cannot be null")
    @Size(min = 2, message = "Name not be less than two characters")
    private String name;

    @NotNull(message = "Password cannot be null")
    @Size(min = 8, message = "Password must be equal or grater than 8 characters")
    private String pwd;
}
```

- UserDto.java
```java
@Data
public class UserDto {
    private String email;
    private String name;
    private String pwd;
    
    private String userId;
    private Date createdAt;
    
    private String encryptedPwd;
}
```

- jpa , modelmapper dependency 추가
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>2.3.8</version>
</dependency>
```

- UserEntity.java
```java
@Data
@Entity
@Table(name = "users")
public class UserEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, length = 50, unique = true)
    private String email;
    @Column(nullable = false, length = 50)
    private String name;
    @Column(nullable = false, unique = true)
    private String userId;
    @Column(nullable = false, unique = true)
    private String encryptedPwd;
}
```

- UserRepository.java
```java
public interface UserRepository extends CrudRepository<UserEntity, Long> {
}
```

- UserService.java (interface)
```java
public interface UserService {
    UserDto createUser(UserDto userDto);
}
```

- UserServiceImpl.java
```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired UserRepository userRepository;

    @Override
    public UserDto createUser(UserDto userDto) {
        userDto.setUserId(UUID.randomUUID().toString());
        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT); //모델메퍼가 변환시킬 수 있는 환경 설정 정보 지정
        UserEntity userEntity = mapper.map(userDto, UserEntity.class);
        userEntity.setEncryptedPwd("encrypted_password");

        userRepository.save(userEntity);
        UserDto returnUserDto = mapper.map(userEntity, UserDto.class);

        return returnUserDto;
    }
}
```

- UserController.java , createUser controller 추가
```java
@RestController
@RequestMapping("/")
public class UserController {
    private Environment env;
    private UserService userService;

    @Autowired
    public UserController(Environment env, UserService userService) {
        this.env = env;
        this.userService = userService;
    }
    @PostMapping("/users")
    public String createUser(@RequestBody RequestUser user) {
        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);
        UserDto userDto = mapper.map(user, UserDto.class);
        userService.createUser(userDto);

        return "Create user method is called";
    }
}
```

- 데이터 베이스 정보 추가
```yaml
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:testdb
```
- user-service 기동 후 127.0.0.1:64585/users , 정상  
```json
{
    "email": "xognstl@naver.com",
    "name": "taehoon",
    "pwd": 1234
}
```

- response 200 => 201 로 변경
- craeteUser 를 하면 반환 시켜줄 vo 객체
```java
@Data
public class ResponseUser {
    private String email;
    private String name;
    private String userId;
}
```
```java
@PostMapping("/users")
public ResponseEntity createUser(@RequestBody RequestUser user) {
        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);
        UserDto userDto = mapper.map(user, UserDto.class);
        userService.createUser(userDto);

        ResponseUser responseUser = mapper.map(userDto, ResponseUser.class);

        return ResponseEntity.status(HttpStatus.CREATED).body(responseUser);
        }
```

- 127.0.0.1:64979/users 실행시 아래와 같이 반환된다.
```json
{
    "email": "xognstl@naver.com",
    "name": "taehoon",
    "userId": "aa4d5cc1-9714-445e-bf0d-649ddd82d895"
}
```


<br>

### 5. Users Microservice - Spring Security 연동
___

- Spring Security
- Authentication(인증) + Authorization(권한)  

1. 애플리케이션에 spring security jar 를 Dependency 에 추가
2. WebSecurityConfigurerAdapter 를 상속받는 Security Configuration 클래스 생성
3. Security Configuration 클래스에 @EnableWebSecurity 추가
4. Authentication -> configure(AuthenticationManagerBuilder auth) 메서드 재정의
5. Password encode 를 위한 BCryptPasswordEncoder 빈 정의
6. Authorization -> configure(HttpSecurity http) 메서드 재정의

- dependency 추가
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

- WebSecurity.java 추가
```java
@Configuration
@EnableWebSecurity // web security 용도로 사용할 것이다.
public class WebSecurity extends WebSecurityConfigurerAdapter {
    @Override   // 권한
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
        http.authorizeRequests().antMatchers("/users/**").permitAll();
        //authorizeRequests : 허용할 수 있는 작업을 제약 , /users/** 를 가져야만 통과
        http.headers().frameOptions().disable(); // 이게 있어야 h2-console 들어갈 수 있음
    }
}
```
- 비밀번호 암호화 작업
- UserServiceImpl
- UserServiceImpl 에서 생성자 주입시 BCryptPasswordEncoder passwordEncoder 어디에서도 생성한적이 없어 Autowired 가 안된다.
- 사용하려면 UserApplication 에 BCryptPasswordEncoder 를 빈으로 등록 해줘야한다.
```java
public class UserServiceApplication {
    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

```java
public class UserServiceImpl implements UserService {
    UserRepository userRepository;
    BCryptPasswordEncoder passwordEncoder;

    //BCryptPasswordEncoder passwordEncoder 어디에서도 생성한적이 없어 Autowired 가 안된다.
    @Autowired
    public UserServiceImpl(UserRepository userRepository, BCryptPasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }
    userEntity.setEncryptedPwd(passwordEncoder.encode(userDto.getPwd()));
}
```

- 127.0.0.1:63431/users => create 하면 비밀번호가 암호화 된것을 확인 할 수있다.
