## 섹션 6. Catalogs and Orders Microservice

### 1. Users Microservice와 Spring Cloud Gateway 연동
___

- user-service API gateway 에 정보 입력
- apigateway-service.application.yml
```yaml
routes:
- id: user-service
  uri: lb://USER-SERVICE
  predicates:
    - Path=/user-service/**
```

```java
@GetMapping("/heath_check")
public String status() {
    return String.format("It's Working in User Service on PORT %s",
            env.getProperty("local.server.port"));
}
```
- http://127.0.0.1:58006/heath_check => 정상 호출
- http://127.0.0.1:8000/user-service/heath_check => 404 , user-service 라는 prefix 추가해주어야함.


<br>

### 2. Users Microservice - 사용자 조회
___

- 전체 사용자 조회, 사용자 정보, 주문내역 조회

- ResponseUser.java 에 주문 정보 추가, UserDto.java 에도 orders 똑같이 추가 
- @JsonInclude : json data에 null 값은 버림. 
```java
@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ResponseUser {
    private String email;
    private String name;
    private String userId;

    private List<ResponseOrder> orders;
}
```

- ResponseOrder.java 추가
```java
@Data
public class ResponseOrder {
    private String productId;
    private Integer qty;
    private Integer unitPrice;  // 단가
    private Integer totalPrice;
    private Date createAt;
    
    private String orderId;
}
```

- UserService.java, 유저 전체 조회, 사용자 조회 함수 추가
```java
public interface UserService {
    UserDto getUserByUserId(String userId);
    Iterable<UserEntity> getUserByAll();
}
```

- UserServiceImpl.java
```java
@Override
public UserDto getUserByUserId(String userId) {
    UserEntity userEntity = userRepository.findByUserId(userId);

    if (userEntity == null) {
        throw new UsernameNotFoundException("User Not Found");
    }

    UserDto userDto = new ModelMapper().map(userEntity, UserDto.class);

    List<ResponseOrder> orders = new ArrayList<>();
    userDto.setOrders(orders);

    return userDto;
}

@Override
public Iterable<UserEntity> getUserByAll() {
    return userRepository.findAll();
}
```

- UserRepository.java
- UserEntity findByUserId(String userId); 추가
```java
public interface UserRepository extends CrudRepository<UserEntity, Long> {
    UserEntity findByUserId(String userId);
}
```

- UserController.java
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

@GetMapping("/users")
public ResponseEntity<List<ResponseUser>> getUsers() {
    Iterable<UserEntity> userList = userService.getUserByAll();

    List<ResponseUser> result = new ArrayList<>();
    userList.forEach(v ->{
        result.add(new ModelMapper().map(v, ResponseUser.class));
    });

    return ResponseEntity.status(HttpStatus.OK).body(result);
}
```

- GET 127.0.0.1:8000/user-service/users/ => User 전체 조회 정상
- GET 127.0.0.1:8000/user-service/users/ebf15443-7217-447a-ba8d-0a81efb58345 => 해당 사용자 조회 가능(주문정보도 List 로 들어있다.)

<br>



