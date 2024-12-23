## 섹션 11. Microservcie간 통신

### 1. Communication types
___

- 동기 방식 : 전체 작업이 마무리 되야 다음 작업으로 넘어가는 방식
- 비동기 방식 : AMQP 방식 사용 헀음
- 클라이언트가 /user-service/users/{userId} 호출을 하면 REST Template OrderService에  
필요한 데이터를 가져오는 방식(orders)

<br>

### 2. RestTemplate
___

- UserServiceApplication.java 에 RestTemplate 빈 등록
```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    @Bean
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }
}
```
- user-service.yml 에 user-service 에서 order-service 에 호출할 url 정보 입력
```yaml
order_service:
  url: http://127.0.0.1:8000/order-service/%s/orders
```
- UserServiceImpl.java 생성자 Environment, RestTemplate 추가 userService -> order-service 구현
```java
@Service
public class UserServiceImpl implements UserService {

    UserRepository userRepository;
    BCryptPasswordEncoder passwordEncoder;
    Environment env;
    RestTemplate restTemplate;
    
    @Autowired
    public UserServiceImpl(UserRepository userRepository, BCryptPasswordEncoder passwordEncoder,
                            Environment env,  RestTemplate restTemplate) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.env = env;
        this.restTemplate = restTemplate;
    }

    @Override
    public UserDto getUserByUserId(String userId) {
        UserEntity userEntity = userRepository.findByUserId(userId);

        if (userEntity == null) {
            throw new UsernameNotFoundException("User Not Found");
        }

        UserDto userDto = new ModelMapper().map(userEntity, UserDto.class);

//        List<ResponseOrder> orders = new ArrayList<>();
        /* using RestTemplate */
        String orderUrl = String.format(env.getProperty("order_service.url"), userId);
        ResponseEntity<List<ResponseOrder>> orderListResponse =
                restTemplate.exchange(orderUrl, HttpMethod.GET, null,   // 호출할 URL, 방식, 파라미터, 받을 방식
                new ParameterizedTypeReference<List<ResponseOrder>>() {
                });
        List<ResponseOrder> orderList = orderListResponse.getBody();
        userDto.setOrders(orderList);

        return userDto;
    }
}
```

- 테스트
  - 회원가입 POST http://127.0.0.1:8000/user-service/users   
  - 주문 POST http://127.0.0.1:8000/order-service/4d37a395-7eae-44eb-9038-12607a750700/orders
```json
{
"productId": "CATALOG-03",
"qty": 5,
"unitPrice": 750    
}
```
  -  GET : http://127.0.0.1:8000/user-service/users/4d37a395-7eae-44eb-9038-12607a750700 

- yaml에 등록된 http://127.0.0.1:8000/order-service/%s/orders url 을 http://microservice_name/URI 로 변경 하는 방법
- RestTemplate 빈등록 하는 부분에 @LoadBalanced 어노테이션만 붙혀주면 됨.
```yaml
  url: http://ORDER-SERVICE/order-service/%s/orders
```

<br>

### 3. FeignClient 사용
___

- Rest 방식을 사용하기 위해 추상화 되어있는 인터페이스를 제공, Spring Cloud Netflix 라이브러리
- 호출하려고 하는 마이크로서비스 http Endpoint에 대한 Interface를 생성
- @FeignClienct 선언
- load balanced 지원

#### FeignClient 적용
- openfeign dependency 추가
- main class에 @EnableFeignClients 어노테이션 추가
- 통신할 마이크로서비스의 호출하고 싶은 함수를 체크하고 인터페이스 등록(order-service 의 getOrder)
```java
@FeignClient(name = "order-service")
public interface OrderServiceClient {

    @GetMapping("/order-service/{userId}/orders")
    List<ResponseOrder> getOrders(@PathVariable String userId);
}
```
- UserServiceImpl 에 금방 만든 인터페이스 생성자 등록
- 인터페이스의 함수만 호출해오면 된다.
```java
@Service
public class UserServiceImpl implements UserService {
    @Override
    public UserDto getUserByUserId(String userId) {
        UserEntity userEntity = userRepository.findByUserId(userId);

        if (userEntity == null) {
            throw new UsernameNotFoundException("User Not Found");
        }
        UserDto userDto = new ModelMapper().map(userEntity, UserDto.class);

        /* using FeignClient */
        List<ResponseOrder> orderList = orderServiceClient.getOrders(userId);
        
        userDto.setOrders(orderList);

        return userDto;
    }

}
```

<br>

### 4. FeignClient 예외 처리
___

- application.yml client 에 로그레벨 DEBUG 설정, main 에 feignClient logger 빈 등록
```yaml
logging:
  level:
    com.example.userservice.client: DEBUG
```
```java
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```
- 이 상태만 가지고 FeignClient 가 호출이 되면 관련 정보 확인 가능
- FeignClient 인터페이스 엔드포인트를 이상한거로 바꿔서 유저를 조회하면 500 에러,  "trace": "feign.FeignException$NotFound: [404] 로그가 뜬다.
- 로그를 추가함으로써 관련 로그 확인 가능 => /orders_ng 잘못 호출된걸 알 수 있다.
```text
[OrderServiceClient#getOrders] ---> GET http://order-service/order-service/db50b431-9f42-4301-b545-a3e8bbf8ffaf/orders_ng HTTP/1.1
[OrderServiceClient#getOrders] ---> END HTTP (0-byte body)
[OrderServiceClient#getOrders] <--- HTTP/1.1 404 (1130ms)
```
- 에러 처리 로직 추가
```java
public class UserServiceImpl implements UserService {
    @Override
    public UserDto getUserByUserId(String userId) {
        UserEntity userEntity = userRepository.findByUserId(userId);

        if (userEntity == null) {
            throw new UsernameNotFoundException("User Not Found");
        }

        UserDto userDto = new ModelMapper().map(userEntity, UserDto.class);
        /* using FeignClient */
        /* Feign exception handing */
        List<ResponseOrder> orderList = null;
        try {
            orderList = orderServiceClient.getOrders(userId);
        } catch (FeignException ex) {
            log.error(ex.getMessage());
        }

        userDto.setOrders(orderList);

        return userDto;
    }
}
```
- 사용자 조회 하면 정상 적으로 조회가 된다. 주문 내용은 에러가 떳기때문에 생략하고 나머지만 출력
```json
{
    "email": "xognstl@naver.com",
    "name": "xognstl",
    "userId": "b8ad97b5-5c60-411b-a73a-e01ef6e8042f"
}
```
```text
[404] during [GET] to [http://order-service/order-service/b8ad97b5-5c60-411b-a73a-e01ef6e8042f/
orders_ng] [OrderServiceClient#getOrders(String)]: 
[{"timestamp":"2024-12-23T01:47:20.537+00:00","status":404,"error":"Not Found",
"message":"No message available",
"path":"/order-service/b8ad97b5-5c60-411b-a73a-e01ef6e8042f/orders_ng"}]
```

<br>

### 5. ErrorDecoder를 이용한 예외 처리
___
- Feign 패키지에서 제공해주는 error decode 인터페이스를 구현시켜서 예외처리
- FeignErrorDecoder.java 생성 후 에러케이스에 따른 처리 추가, main 에 빈도 등록 해야한다.
```java
public class FeignErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        switch (response.status()) {
            case 400:
                break;
            case 404:
                if (methodKey.contains("getOrders")) {
                    return new ResponseStatusException(HttpStatus.valueOf(response.status()),
                            "User's orders is empty");
                }
                break;
            default:
                return new Exception(response.reason());
            
        }
        return null;
    }
}
```
- FeignErrorDecoder class 는 에러가 발생할 때 errorDecode 가 사용될 것이기 때문에 따로 주입 X 
```java
public class UserServiceImpl implements UserService {
    @Override
    public UserDto getUserByUserId(String userId) {
        UserEntity userEntity = userRepository.findByUserId(userId);
        if (userEntity == null) {
            throw new UsernameNotFoundException("User Not Found");
        }
        UserDto userDto = new ModelMapper().map(userEntity, UserDto.class);
        /* ErrorDecoder */
        List<ResponseOrder> orderList = orderServiceClient.getOrders(userId);

        userDto.setOrders(orderList);

        return userDto;
    }
}
```
- status 404 로 변경 된 것을 확인 할 수 있다.
```text
"timestamp": "2024-12-23T02:06:30.023+00:00",
"status": 404,
"error": "Not Found",
"message": "User's orders is empty", <= 입력 했던 메시지 확인 가능!
```
- message yaml 로 빼기
```yaml
order_service:
  url: http://ORDER-SERVICE/order-service/%s/orders
  exception:
    order_is_empty: User's orders is empty
```
- env를 사용 하려면 Component 로 등록을 해야하기 때문에 main class의 FeignErrorDecoder 부분 주석 처리.
```java
@Component
public class FeignErrorDecoder implements ErrorDecoder {
    Environment env;

    @Autowired
    public FeignErrorDecoder(Environment env) {
        this.env = env;
    }

    @Override
    public Exception decode(String methodKey, Response response) {
        switch (response.status()) {
            case 400:
                break;
            case 404:
                if (methodKey.contains("getOrders")) {
                    return new ResponseStatusException(HttpStatus.valueOf(response.status()),
                            env.getProperty("order_service.exception.orders_is_empty"));
                }
                break;
            default:
                return new Exception(response.reason());

        }
        return null;
    }
}
``` 
<br>

### 6. 데이터 동기화 문제
___
- 하나의 마이크로서비스를 하나 이상의 인스턴스에서 기동 시켰을때 DB도 각각 가지고 있다고 하면 동기화문제
- 주문을 여러개 하면 첫째 인스턴스에 저장되고 2번째 주문은 2번째 인스턴스에 저장되는 상황이 나올 수 있다.
- Feign Client 나 restTemplate 으로 데이터 요청할때 DB가 나눠저서 자료가 저장되어있으니 어딜 조회 해야할까?
1. DB를 나눠서 사용하는게 아니라 한개의 DB를 사용 하는 방법
2. DB를 각각 사용하고 동기화 시키는 방법, 각각 DB에 저장 하는게 아니라 메시지 큐잉서버 이용
3. 데이터 -> 메시지 큐잉 서버 1개 -> DB 1개 

- 실제로 인스턴스 2개 키고 주문 5개 한결과 1번 인스턴스 2개, 2번 인스턴스 3개가 저장 되었다.
- 오더를 조회하면 2개 ,3개 번갈아가면서 나온다.
