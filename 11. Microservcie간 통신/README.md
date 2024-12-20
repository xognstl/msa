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
