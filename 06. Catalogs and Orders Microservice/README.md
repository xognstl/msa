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
@GetMapping("/users")
public ResponseEntity<List<ResponseUser>> getUsers() {
    Iterable<UserEntity> userList = userService.getUserByAll();

    List<ResponseUser> result = new ArrayList<>();
    userList.forEach(v ->{
        result.add(new ModelMapper().map(v, ResponseUser.class));
    });

    return ResponseEntity.status(HttpStatus.OK).body(result);
}

@GetMapping("/users/{userId}")
public ResponseEntity<ResponseUser> getUser(@PathVariable("userId") String userId) {
    UserDto userDto = userService.getUserByUserId(userId);

    ResponseUser returnValue = new ModelMapper().map(userDto, ResponseUser.class);

    return ResponseEntity.status(HttpStatus.OK).body(returnValue);
}
```

- GET 127.0.0.1:8000/user-service/users/ => User 전체 조회 정상
- GET 127.0.0.1:8000/user-service/users/ebf15443-7217-447a-ba8d-0a81efb58345 => 해당 사용자 조회 가능(주문정보도 List 로 들어있다.)

<br>

### 3. Catalogs Microservice 
___

- 사용자가 주문하기 전에 상품 목록을 검색하기 위한 용도

- dependencies : devTools, Lombok, Web, JPA, Eureka Discovery Client, h2, modelmapper
- ddl-auto : create-drop 기동시 자동 데이터 등록
```yaml
jpa:
  hibernate:
    ddl-auto: create-drop
  show-sql: true
  generate-ddl: true
```
- resource.data.sql 생성 및 insert 문 입력
- CatalogEntity.java
```java
@Data
@Entity
@Table(name = "catalog")
public class CatalogEntity implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120, unique = true)
    private String productId;
    @Column(nullable = false)
    private String productName;
    @Column(nullable = false)
    private Integer stock;
    @Column(nullable = false)
    private Integer unitPrice;

    @Column(nullable = false, updatable = false, insertable = false)
    @ColumnDefault(value = "CURRENT_TIMESTAMP")
    private Date createAt;
}
```

- CatalogDto.java
```java
@Data
public class CatalogDto implements Serializable {
    private String productId;
    private Integer qty;
    private Integer unitPrice;
    private Integer totalPrice;

    private String orderId;
    private String uerId;
}
```

- ResponseCatalog.java
```java
@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ResponseCatalog {
    private String productId;
    private String productName;
    private Integer unitPrice;
    private Integer stock;
    private Date createdAt;

}
```

- CatalogRepository.java
```java
public interface CatalogRepository extends CrudRepository<CatalogEntity, Long> {
    CatalogEntity findByProductId(String productId);
}
```

- CatalogService.java
```java
public interface CatalogService {
    Iterable<CatalogEntity> getAllCatalogs();
}
```

- CatalogServiceImpl.java
```java
@Data
@Slf4j
@Service
public class CatalogServiceImpl implements CatalogService {
    CatalogRepository catalogRepository;

    @Autowired
    public CatalogServiceImpl(CatalogRepository catalogRepository) {
        this.catalogRepository = catalogRepository;
    }

    @Override
    public Iterable<CatalogEntity> getAllCatalogs() {
        return catalogRepository.findAll();
    }
}
```

- CatalogController.java
```java
@RestController
@RequestMapping("/catalog-service")
public class CatalogController {
    Environment env;
    CatalogService catalogService;

    @Autowired
    public CatalogController(Environment env, CatalogService catalogService) {
        this.env = env;
        this.catalogService = catalogService;
    }

    @GetMapping("/heath_check")
    public String status() {
        return String.format("It's Working in Catalog Service on PORT %s",
                env.getProperty("local.server.port"));
    }

    @GetMapping("/catalogs")
    public ResponseEntity<List<ResponseCatalog>> getCatalogs() {
        Iterable<CatalogEntity> catalogList = catalogService.getAllCatalogs();

        List<ResponseCatalog> result = new ArrayList<>();
        catalogList.forEach(v ->{
            result.add(new ModelMapper().map(v, ResponseCatalog.class));
        });

        return ResponseEntity.status(HttpStatus.OK).body(result);
    }
}
```

- gataway server에 catalog 등록
```yaml
- id: catalog-service
  uri: lb://CATALOG-SERVICE
  predicates:
    - Path=/catalog-service/**
```

- GET : 127.0.0.1:8000/catalog-service/catalogs => catalog 에 insert 된 3개의 상품조회가능


<br>

### 4. Catalogs Microservice
___

- 사용자가 직접 주문하는 메소드, 주문별 내역조회 
- dependencies : devTools, Lombok, Web, JPA, Eureka Discovery Client, h2, modelmapper
- Serializable : 가지고 있는 Object 객체를 다른 네트워크로 전송하거나 DB 보관을 위해 마셜링, 언마셜링 작업하기 위해 사용

- OrderEntity.java
```java
@Data
@Entity
@Table(name="orders")
public class OrderEntity implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120, unique = true)
    private String productId;
    @Column(nullable = false)
    private Integer qty;
    @Column(nullable = false)
    private Integer unitPrice;
    @Column(nullable = false)
    private Integer totalPrice;

    @Column(nullable = false)
    private String userId;
    @Column(nullable = false, unique = true)
    private String orderId;

    @Column(nullable = false, updatable = false, insertable = false)
    @ColumnDefault(value = "CURRENT_TIMESTAMP")
    private Date createdAt;
}
```

- OrderDto.java
```java
@Data
public class OrderDto implements Serializable {
    private String productId;
    private Integer qty;
    private Integer unitPrice;
    private Integer totalPrice;

    private String orderId;
    private String userId;
}
```

- ResponseOrder.java
```java
@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ResponseOrder {
    private String productId;
    private Integer qty;
    private Integer unitPrice;
    private Integer totalPrice;
    private Date createdAt;

    private String orderId;

}
```
- RequestOrder.java
```java
@Data
public class RequestOrder {
    private String productId;
    private Integer qty;
    private Integer unitPrice;
}
```

- OrderRepository.java
```java
public interface OrderRepository extends CrudRepository<OrderEntity, Long> {
    OrderEntity findByOrderId(String orderId);

    Iterable<OrderEntity> findByUserId(String userId);
}
```

- OrderService.java
```java
public interface OrderService {
    OrderDto createOrder(OrderDto orderDetails);
    OrderDto getOrderByOrderId(String orderId);
    Iterable<OrderEntity> getOrdersByUserId(String userId);
}
```

- OrderServiceImpl.java
```java
@Service
public class OrderServiceImpl implements OrderService {

    OrderRepository orderRepository;

    @Autowired
    public OrderServiceImpl(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public OrderDto createOrder(OrderDto orderDto) {
        orderDto.setOrderId(UUID.randomUUID().toString());
        orderDto.setTotalPrice(orderDto.getQty() * orderDto.getUnitPrice());

        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);
        OrderEntity orderEntity = mapper.map(orderDto, OrderEntity.class);

        orderRepository.save(orderEntity);

        OrderDto returnValue = mapper.map(orderEntity, OrderDto.class);

        return returnValue;
    }

    @Override
    public OrderDto getOrderByOrderId(String orderId) {
        OrderEntity orderEntity = orderRepository.findByOrderId(orderId);
        OrderDto orderDto = new ModelMapper().map(orderEntity, OrderDto.class);
        return orderDto;
    }

    @Override
    public Iterable<OrderEntity> getOrdersByUserId(String userId) {
        return orderRepository.findByUserId(userId);
    }
}
```

- OrderController.java
```java
@RestController
@RequestMapping("/order-service")
public class OrderController {
    Environment env;
    OrderService orderService;

    @Autowired
    public OrderController(Environment env, OrderService orderService) {
        this.env = env;
        this.orderService = orderService;
    }

    @GetMapping("/heath_check")
    public String status() {
        return String.format("It's Working in Order Service on PORT %s",
                env.getProperty("local.server.port"));
    }

    @PostMapping("/{userId}/orders")
    public ResponseEntity<ResponseOrder> createOrder(@PathVariable("userId") String userId,
                                                     @RequestBody RequestOrder orderDetails) {
        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);

        OrderDto orderDto = mapper.map(orderDetails, OrderDto.class);

        orderDto.setUserId(userId);
        OrderDto createOrder = orderService.createOrder(orderDto);

        ResponseOrder responseOrder = mapper.map(createOrder, ResponseOrder.class);

        return ResponseEntity.status(HttpStatus.CREATED).body(responseOrder);
    }

    @GetMapping("/{userId}/orders")
    public ResponseEntity<List<ResponseOrder>> getOrder(@PathVariable("userId") String userId) {
        Iterable<OrderEntity> orderList = orderService.getOrdersByUserId(userId);
        List<ResponseOrder> result = new ArrayList<>();
        orderList.forEach(v -> {
            result.add(new ModelMapper().map(v, ResponseOrder.class));
        });

        return ResponseEntity.status(HttpStatus.OK).body(result);
    }
}
```

- api gateway 에 order service 등록

- 테스트
- POST : 127.0.0.1:8000/user-service/users 유저 등록
- GET : 127.0.0.1:8000/catalog-service/catalogs 전체 상품 리스트
- POST : 127.0.0.1:8000/order-service/7cb28304-f9b0-4892-9298-23df5a8a9c2e/orders 주문
```json
{
  "productId": "CATALOG-001",
  "qty": 10,
  "unitPrice": 1500
}
```
=> 결과
```json
{
  "productId": "CATALOG-001",
  "qty": 10,
  "unitPrice": 1500,
  "totalPrice": 15000,
  "orderId": "dc56ff69-e19d-4dc2-b012-458164db476a"
}
```
- GET : 127.0.0.1:8000/order-service/7cb28304-f9b0-4892-9298-23df5a8a9c2e/orders 주문 정보 확인


