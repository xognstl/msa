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

