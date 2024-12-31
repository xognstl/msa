## 섹션 14. 장애 처리와 Microservice 분산 추적

### 1. CircuitBreaker와 Resilience4J의 사용
___
- 예를 들어 UserService getUser() 호출(Feign Client) -> OrderService getOrders() -> CatalogService getCatalogs()  
이런식으로 MicroService 가 연결 되어있을때 OrderService 나 CatalogService 에서 오류가 발생해도 UserService 자체가 500 에러 날  
때가 있다. 
- Order, Catalog 에서 문제가 생겨도 Feign Client 쪽에서는 에러가 발생 했을때 에러를 대신할 수 있는 디폴트값이나  
우회 할 수 있는 또는 정상적인 뎅치터처럼 보여줄 수 있는 다른 데이터를 보여주는게 UserService 에 준비가 되어있어야한다.
- 문제가 생겼을때 해당 function 을 막아주고 문제가 생겼던 서비스를 재사용할 수 있는 상태로 복구가 된다고 하면,  
이전에 사용했던 것 처럼 정상적인 흐름으로 바꿔주는 장치를 CircuitBreaker 라고 한다.

- CircuitBreaker
  - 장애가 발생하는 서비스에서 반복적인 호출이 되지 못하게 차단
  - 특정 서비스가 정상적으로 동작하지 않을 경우 다른 기능으로 대체 수행(장애 회피)
  - User <-> Order 통신에 아무 문제가 없으면 Circuit Breaker 가 Closed 상태 User <-> Circuit Breaker Close <-> Order 
  - 통신에 문제가 생기면 User <-> Circuit Breaker Open <-> Order
  - User 에서 Order를 사용함에 있어 접속이 안되거나 다른이유로 정상 서비스가 불가하면 그 수치가 일정수치 이상   
(ex. 30초에 10번 호출 응답 X) => Circuit Breaker Open 가되고 Circuit Breaker 자체적으로 기본값, 우회 할 수 있는 값 return 

- Spring Cloud Ntflix Htstrix 사용(2.4 이전 버전) => Resilience4J 사용
- Htstrix Dashboard/Turbine => Micrometer + Monitoring System 사용
- Resilience4J 는 Circuitbreaker, ratelimiter, bulkhead, retry, timelimiter, cache 기능 사용
- 에러가 발생을 했을때 정상적인 서비스 처럼 가용할 수 있는 처리를 해주는 라이브러리

<br>

### 2. Users Microservice에 CircuitBreaker 적용
___
- 회원가입 후 GET 127.0.0.1:8000/user-service/users/6bdc4fb9-9b3d-45f5-ad40-ca5b0768bf1d  
=> OrderService 기동하지 않으면 500 에러 발생, java.net.UnknownHostException: order-service
- resilience4j dependency 추가
- OrderServiceImpl : circuitBreakerFactory 주입 (Default 값의 circuitBreakerFactory 사용 가능)
```java
@Service
@Slf4j
public class UserServiceImpl implements UserService {
    CircuitBreakerFactory circuitBreakerFactory;
    
    @Autowired
    public UserServiceImpl(UserRepository userRepository, BCryptPasswordEncoder passwordEncoder,
                            Environment env,  RestTemplate restTemplate, OrderServiceClient orderServiceClient,
                           CircuitBreakerFactory circuitBreakerFactory) {
        this.circuitBreakerFactory = circuitBreakerFactory;
    }


    @Override
    public UserDto getUserByUserId(String userId) {
        UserEntity userEntity = userRepository.findByUserId(userId);

        if (userEntity == null) {
            throw new UsernameNotFoundException("User Not Found");
        }

        UserDto userDto = new ModelMapper().map(userEntity, UserDto.class);
        /* ErrorDecoder */
//        List<ResponseOrder> orderList = orderServiceClient.getOrders(userId);
        /* Circuit Breaker */
        CircuitBreaker circuitbreaker = circuitBreakerFactory.create("circuitbreaker"); // circuit breaker 생성
        List<ResponseOrder> orderList = circuitbreaker.run(() -> orderServiceClient.getOrders(userId),
                throwable -> new ArrayList<>()); // 문제가 발생하면 빈 ArrayList 반환

        userDto.setOrders(orderList);

        return userDto;
    }

}
```
- 127.0.0.1:8000/user-service/users/737b5e08-9456-4fee-905d-8ef382e5fe08 다시 하면 아래와 같은 결과가 나온다.  
에러는 동일 하게 콘솔에 찍힌다.
```json
{
    "email": "xognstl@naver.com",
    "name": "xognstl",
    "userId": "737b5e08-9456-4fee-905d-8ef382e5fe08",
    "orders": []
}
```
- 사용자 커스텀
  - Resilience4JCircuitBreakerFactory 커스터마이징 할때 사용
  - failureRateThreshold(4) : Circuit Breaker 를 열것인지 닫을 것인지 수치 , 실패확률 , default 50
  - waitDurationInOpenState : Circuit Breaker open 상태 지속기간 (1)초가 지나면 다시 close , default 60초
  - slidingWindowType : circuit Breaker 가 닫힐때 지금까지 호출했었던 결과 값을 count기반 횟수, 시간기반일지 지정 가능, default count
  - slidingWindowSize : close 될때 결과 값을 기록하기 위해 어느정도 수치를 가지고 저장할지
  - timeoutDuration : supplier 가 어느정도 문제가 생겼을때 오류로 간주해 open 할지 정하는 것(4초동안 응답이 없을때 open )
```java
@Configuration
public class Resilience4JConfig {

    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> globalCustomConfiguration() {

        CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
                .failureRateThreshold(4)
                .waitDurationInOpenState(Duration.ofMillis(1000))
                .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
                .slidingWindowSize(2)
                .build();

        TimeLimiterConfig timeLimiterConfig = TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(4))
                .build();

        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
                .timeLimiterConfig(timeLimiterConfig)
                .circuitBreakerConfig(circuitBreakerConfig)
                .build()
                );
    }
}
```
- 테스트를 위해 kafka, mariaDB는 잠시 주석 => jpa, h2 로 변경

<br>

### 3. 분산 추적의 개요 Zipkin 서버 설치
___
- Zipkin
  - twitter에서 사용하는 분산 환경의 Timing 데이터 수집, 추적 시스템(오픈소스)
  - 분산환경에서의 시스템 병목 현상 파악
  - Collector, Query Service, Database WebUI 로 구성
  - span : 하나의 요청에 사용되는 작업의 단위 , 64 bit unique Id
  - trace : 트리 구조로 이루어진 span , 하나의 요청에 대한 같은 Trace Id 발급
  - 사용자가 가지고 있는 마이크로 서비스는 모든 정보를 zipkin 에 데이터를 전달, 사용자 서비스에서 다른 서비스를 호출 했을 때   
  그 정보도 zipkin 서비스로 전달


- Spring Cloud Sleuth
  - 스프링 부트 애플리케이션을 Zipkin 과 연동, 요청 값에 따른 Trace ID, Span ID 부여
  - Trace와 Span ID를 로그에 추가 가능  
  (Servlet filter, rest template, scheduled actions, message channels, feign client 에서 ID 발생)
  - Trace ID, Span ID로 만들어진 내용을 zipkin서버에 전달(sleuth), 누적되어진 데이터 값을 시각화(zipkin)


- Spring Cloud Sleuth + Zipkin  
  <img src="https://github.com/user-attachments/assets/aeb78036-ac36-40ac-9558-a84ec1f2a387" width="500" height="300">
- 사용자의 요청이 시작되고 끝날 때까지 같은 TraceID 사용, 그 사이의 필요한 서비스 간의 트랜잭션들이 발생하면  
Span ID 가 발생 된다.


- Circuit Breaker 는 작동할 수 없는 상태를 우회할 수 있는 방법이고, Slueth , zipkin 으로 마이크로서비스가  
연결되어 있는 상태 값을 추적해서 누가 누구를 호출했고 시간이 얼마나 걸렸고 정상, 비정상인지 알려주고  
시각화 할 수 있는 도구


- Zipkin 설치
- window 에서 git bash 로 해당 명령어 사용 가능 , cmd 에선 bash 따로 뭐 해줘야함.
```text
curl -sSL https://zipkin.io/quickstart.sh | bash -s //다운
java -jar zipkin.jar    // 실행
```
- https://zipkin.io/pages/quickstart.html 사이트의 latest release 로도 다운 가능
- http://127.0.0.1:9411/zipkin/ 로 dashboard 접속

<br>

### 4. Spring Cloud Sleuth + Zipkin을 이용한 Microservice의 분산 추적
___
- zipkin + sleuth dependency 추가
```xml
<!-- zipkin -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-zipkin</artifactId>
<version>2.2.3.RELEASE</version>
</dependency> 
```
- zipkin Server 위치 지정
```yaml
spring:
  application:
    name: user-service
  zipkin:
    base-url: http://127.0.0.1:9411
    enabled: true
  sleuth:
    sampler:
      probability: 1.0
```
- orderService 호출 전후로 log
```java
/* Circuit Breaker */
log.info("Before call orders microservice");
CircuitBreaker circuitbreaker = circuitBreakerFactory.create("circuitbreaker"); // circuit breaker 생성
List<ResponseOrder> orderList = circuitbreaker.run(() -> orderServiceClient.getOrders(userId),
      throwable -> new ArrayList<>()); // 문제가 발생하면 빈 ArrayList 반환
log.info("After call orders microservice");
```
- OrderService 에도 dependency , application.yaml 설정 똑같이 추가
- OrderService 에서 사용자 요청이 들어 왔을때 그걸 처리해주는 함수에 동일하게 log 출력
```java
    @PostMapping("/{userId}/orders")
public ResponseEntity<ResponseOrder> createOrder(@PathVariable("userId") String userId,
@RequestBody RequestOrder orderDetails) {
        log.info("Before add orders data");
        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);

        OrderDto orderDto = mapper.map(orderDetails, OrderDto.class);
        orderDto.setUserId(userId);
        /* jpa */
        OrderDto createOrder = orderService.createOrder(orderDto);
        ResponseOrder responseOrder = mapper.map(createOrder, ResponseOrder.class);
        log.info("After add orders data");
        return ResponseEntity.status(HttpStatus.CREATED).body(responseOrder);
        }
        
@GetMapping("/{userId}/orders")
public ResponseEntity<List<ResponseOrder>> getOrder(@PathVariable("userId") String userId) {
    log.info("Before call retrieve orders data");
    Iterable<OrderEntity> orderList = orderService.getOrdersByUserId(userId);
    List<ResponseOrder> result = new ArrayList<>();
    orderList.forEach(v -> {
        result.add(new ModelMapper().map(v, ResponseOrder.class));
    });
    log.info("Add call retrieve orders data");

    return ResponseEntity.status(HttpStatus.OK).body(result);
}
```
- test 
- 회원가입 -> 상품 주문
- POST 127.0.0.1:8000/order-service/4bdbcc52-562f-4a2b-aea4-862d70909a91/orders 2건 주문
```text
2024-12-31 15:11:40.201  INFO [order-service,79a6c74be9dc42d4,79a6c74be9dc42d4] : Before add orders data
2024-12-31 15:11:40.476  INFO [order-service,79a6c74be9dc42d4,79a6c74be9dc42d4] : After add orders data
2024-12-31 15:11:45.737  INFO [order-service,1f493826d156a2d1,1f493826d156a2d1] : Before add orders data
2024-12-31 15:11:45.741  INFO [order-service,1f493826d156a2d1,1f493826d156a2d1] : After add orders data
traceID, spanID 발급 확인
```

<img src="https://github.com/user-attachments/assets/20b73929-dc2b-438c-a5d5-00f9bd2a1faa" width="700" height="300">


- zipkin dashboard 에서 해당 traceID로 검색하면 위와같은 정보를 볼 수 있다.

- 사용자 상세정보 보기 
```text
userService 로그
2024-12-31 15:17:47.140  INFO [user-service,f57bf85002e246c6,f57bf85002e246c6] c.e.userservice.service.UserServiceImpl  : Before call orders microservice
2024-12-31 15:17:47.204 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] ---> GET http://order-service/order-service/4bdbcc52-562f-4a2b-aea4-862d70909a91/orders HTTP/1.1
2024-12-31 15:17:47.205 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] ---> END HTTP (0-byte body)
2024-12-31 15:17:47.801 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] <--- HTTP/1.1 200 (596ms)
2024-12-31 15:17:47.802 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] connection: keep-alive
2024-12-31 15:17:47.802 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] content-type: application/json
2024-12-31 15:17:47.802 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] date: Tue, 31 Dec 2024 06:17:47 GMT
2024-12-31 15:17:47.802 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] keep-alive: timeout=60
2024-12-31 15:17:47.803 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] transfer-encoding: chunked
2024-12-31 15:17:47.803 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] 
2024-12-31 15:17:47.803 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] [{"productId":"CATALOG-001","qty":5,"unitPrice":1500,"totalPrice":7500,"createdAt":"2024-12-31T06:11:40.368+00:00","orderId":"4d532ad3-b802-4cf5-a5d3-dc1db68a8165"},{"productId":"CATALOG-002","qty":10,"unitPrice":1500,"totalPrice":15000,"createdAt":"2024-12-31T06:11:45.739+00:00","orderId":"072197ba-709a-4047-b772-ea055d18ee09"}]
2024-12-31 15:17:47.804 DEBUG [user-service,f57bf85002e246c6,02ba6b5fc59dfd1e] c.e.u.client.OrderServiceClient          : [OrderServiceClient#getOrders] <--- END HTTP (331-byte body)
2024-12-31 15:17:47.843  INFO [user-service,f57bf85002e246c6,f57bf85002e246c6] c.e.userservice.service.UserServiceImpl  : After call orders microservice
중간에 Feign Client 를 통해서 OrderService 호출 할때 새로운 span ID 발급도 확인 가능

OrderService 로그
2024-12-31 15:17:47.591  INFO [order-service,f57bf85002e246c6,7aa027e2b00b2895] c.e.o.controller.OrderController         : Before call retrieve orders data
2024-12-31 15:17:47.787  INFO [order-service,f57bf85002e246c6,7aa027e2b00b2895] c.e.o.controller.OrderController         : Add call retrieve orders data
OrderService 와 UserService 의 traceID 가 같은 것을 확인 할 수 있다. => 같은 요청이라는 것을 알 수 있다.
```

<img src="https://github.com/user-attachments/assets/272ca4a7-2e4f-4f7b-b8f5-e8e45d1ef021" width="700" height="300">

- 마이크로 서비스들이 서로 호출을 함에 있어서 발생되었던 순차적인 흐름들, 데이터 호출 관계에 대해 정확하게  
데이터의 이력을 보관하고 있는 트레이싱 하고 있는 역할을 zipkin Server 가 해주고  
스프링 부트에서 발생했던 로그 데이터 값을 zipkin Server 로 전달시켜주는 역할을 sleuth 라이브러리로 처리 했다.


- zipkin dashboard 에서 find a trace 메뉴에서는 해당 서비스 이름으로 부여되어있는 다양한 정보를 알 수있다.
- dependencies 에서는 시간의 범위를 지정하면 user-> order 호출되었던 관계를 시각화해서 볼 수 있다.
 
- orderservice 장애 내기
```java
@GetMapping("/{userId}/orders")
public ResponseEntity<List<ResponseOrder>> getOrder(@PathVariable("userId") String userId) throws Exception {
    log.info("Before call retrieve orders data");
    Iterable<OrderEntity> orderList = orderService.getOrdersByUserId(userId);
    List<ResponseOrder> result = new ArrayList<>();
    orderList.forEach(v -> {
        result.add(new ModelMapper().map(v, ResponseOrder.class));
    });
    try {
        Thread.sleep(1000);
        throw new Exception(" 장애 발생 ");
    } catch (InterruptedException exception) {
        log.warn(exception.getMessage());
    }
        
    log.info("Add call retrieve orders data");

    return ResponseEntity.status(HttpStatus.OK).body(result);
}
```
- 조회 하면 아래와 같이 나온다.
```json
{
    "email": "xognstl@naver.com",
    "name": "xognstl",
    "userId": "4bdbcc52-562f-4a2b-aea4-862d70909a91",
    "orders": []
}
```
- userService 에 찍힌 trace ID로 zipkin 에 검색
  <img src="https://github.com/user-attachments/assets/ced29086-6b0a-42af-aa6d-4bb10688f4a6" width="700" height="300">

- zipkin 을 이용하여 많은 마이크로 서비스가 연결되어 있을 때 중간에 문제가 생기는게 있는지, 어디서 병목 현상이 일어나는지 유추 할 수 있다.


