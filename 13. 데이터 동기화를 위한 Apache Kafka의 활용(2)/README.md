## 섹션 13. 데이터 동기화를 위한 Apache Kafka의 활용 (2)

### 1. Orders Microservice와 Catalogs Microservice에 Kafka Topic의 적용
___
- Order Service -> Catalog Service 에 데이터를 Kafka 사용(상품 수량 업데이트)
- 현재 microService 의 DB는 각각 독립적으로 구성 해놓음
- Order Service 에 주문 정보, 주문했던 상품 수량 정보가 들어가고 Catalog Service 의 재고 수량도 - 해줘야한다. (동기화 필요)
- Order Service 에서 Kafka Topic 으로 메시지 전송(Producer)
- Catalog Service 에서 Kafka Topic 에 전송 된 메시지 취득 (Consumer)

<br>

### 2. Catalogs Microservice 수정
___
- kafka dependency 등록
- KafkaConsumerConfig.java(빈등록)
  - ConsumerFactory : 토픽에 접속하기 위한 접속 정보
  - ConcurrentKafkaListenerContainerFactory : 토픽에 어떤 변경 사항이 있는지 리스닝하는 리스너를 등록
  - Topic의 값이 json 포멧 이기 때문에 역으로 해석해서 사용하기 때문에 deserializer 사용
```java
@EnableKafka
@Configuration
public class KafkaConsumerConfig {
    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> properties = new HashMap<>();

        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9092");
        // 카프카에서 토픽에 쌓여있는 메시지를 가져가는 컨슈머 들을 그룹핑
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "consumerGroupId"); 
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        return new DefaultKafkaConsumerFactory<>(properties);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory
                = new ConcurrentKafkaListenerContainerFactory<>();
        kafkaListenerContainerFactory.setConsumerFactory(consumerFactory());    // 위에 정보 등록

        return kafkaListenerContainerFactory;
    }
}
```
- KafkaConsumer class
  - 토픽에 변경된 데이터 값을 실제 DB에 반영 해주는 작업
  - @KafkaListener 를 가지고 있는 메소드에서 값이 변경되면 topic 에서 캐치해서 수량 update 해주는 로직을 작성
```java
@Service
@Slf4j
public class KafkaConsumer {    // 리스너를 이용해 데이터를 가져오고 그 값으로 Db에 있는 자료를 update

    CatalogRepository repository;

    @Autowired
    public KafkaConsumer(CatalogRepository repository) {
        this.repository = repository;
    }

    @KafkaListener(topics = "example-catalog-topic")
    public void updateQty(String kafkaMessage) {
        log.info("Kafka Message = " + kafkaMessage);
        Map<Object, Object> map = new HashMap<>();
        ObjectMapper mapper = new ObjectMapper();
        try {
            map = mapper.readValue(kafkaMessage, new TypeReference<Map<Object, Object>>() {});
            // Kafka 메시지가 String 으로 들어오지만 json 으로 변경
        } catch (JsonProcessingException ex) {
            ex.printStackTrace();
        }

        CatalogEntity entity = repository.findByProductId((String) map.get("productId"));
        if (entity != null) {
            entity.setStock(entity.getStock() - (Integer)map.get("qty"));
            repository.save(entity);
        }
    }
}
```

<br>

### 3. Orders Microservice 수정
___
- 빈등록
  - ProducerFactory
  - kafka template : topic 에 데이터를 보내기 위해서 사용되는 객체
```java
@EnableKafka
@Configuration
public class KafkaProducerConfig {
    //Kafka의 정보가 들어가있는 곳
    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> properties = new HashMap<>();

        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9092");
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        return new DefaultKafkaProducerFactory<>(properties);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```
- KafkaProducer class
  - kafka template 가져와서 send 라는 동작 실행
  - 주문을 json 형태로 변경하여 kafka로 전달 하는 로직 작성
```java
@Service
@Slf4j
public class KafkaProducer {

    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    public KafkaProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public OrderDto send(String topic, OrderDto orderDto) {
        // 주문이 들어오면 JSON 포멧으로 변경
        ObjectMapper mapper = new ObjectMapper();
        String jsonInString = "";
        try {
            jsonInString = mapper.writeValueAsString(orderDto);
        } catch (JsonProcessingException exception) {
            exception.printStackTrace();
        }

        kafkaTemplate.send(topic, jsonInString);
        log.info("Kafka Producer sent data from the Order microService : " + orderDto);

        return orderDto;
    }

}
```
- 주문하는 Controller 수정
  - Kafka message 전달하는 코드 추가
```java
@RestController
@RequestMapping("/order-service")
public class OrderController {
    Environment env;
    OrderService orderService;
    KafkaProducer kafkaProducer;

    @Autowired
    public OrderController(Environment env, OrderService orderService, KafkaProducer kafkaProducer) {
        this.env = env;
        this.orderService = orderService;
        this.kafkaProducer = kafkaProducer;
    }
    @PostMapping("/{userId}/orders")
    public ResponseEntity<ResponseOrder> createOrder(@PathVariable("userId") String userId,
                                                     @RequestBody RequestOrder orderDetails) {
        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);

        OrderDto orderDto = mapper.map(orderDetails, OrderDto.class);
        /* jpa */
        orderDto.setUserId(userId);
        OrderDto createOrder = orderService.createOrder(orderDto);
        ResponseOrder responseOrder = mapper.map(createOrder, ResponseOrder.class);

        /* send this order to the kafka */
        kafkaProducer.send("example-catalog-topic", orderDto);

        return ResponseEntity.status(HttpStatus.CREATED).body(responseOrder);
    }

}
```

<br>

### 4. Kafka를 활용한 데이터 동기화 테스트 (1)
___
- zookeeper, kafka, eureka, api-gateway, config, order, catalog 기동
- user-service 기동 X 임의의 사용자 아이디로 테스트 가능 하게 되어있다.
- POST 127.0.0.1:8000/order-service/804b813d-3fa4-4cfc-8ee8-6ab23a8417bc/orders : 주문
```json
{
    "productId": "CATALOG-003",
    "qty": 5,
    "unitPrice": 2000
}
```
- GET 127.0.0.1:8000/catalog-service/catalogs 보면 qty 가 5개 준것을 확인 할 수 있다.  

<br>

### 5. Multi Orders Microservice 사용에 대한 데이터 동기화 문제
___
- Order-Service 2개 기동 -> users의 요청 분산처리 , orders의 데이터도 분산저장 : 데이터 동기화 문제
- 로그인 후 주문을 4개한 결과 1번 order-service 에 2개 , 2번 order-service에 2개의 주문이 저장되었다.
- 주문확인 GET 127.0.0.1:8000/user-service/users/b785de1e-5b78-4ab9-a3aa-9f226830c129 을 검색하면 주문이 2개씩 조회가된다.
- 

<br>

### 6. Kafka Connect를 활용한 단일 데이터베이스를 사용
___
- Order Service 에 요청된 주문 정보를 DB가 아니라 Kafka Topic으로 전송
- Kafka Topic에 설정 된 Kafka Sink Connect를 사용해 단일 DB에 저장 -> 데이터 동기화
- order,catalog 에 대해서 수량 업데이트 하는 부분은 orderService에서 Producer를 만든 다음 전송하는 부분은 문제가 없다.
- 카프카 서버에 저장되어진 토픽의 데이터 값을 체크해서 그 데이터를 단일 DB에 반영시켜주는 역할을 Sink Connector 사용

<br>

### 7. Orders Microservice 수정 - MariaDB
___
- OrderService db h2 -> mariaDB로 변경
```yaml
  datasource:
    driver-class-name: org.mariadb.jdbc.Driver
    url: jdbc:mariadb://localhost:3306/mydb
    username: root
    password: test1234
#    driver-class-name: org.h2.Driver
#    url: jdbc:h2:mem:testdb
```
- 주문을 실행하면 mariaDB의 orders 테이블에 데이터가 쌓인다

<br>

### 8. Orders Microservice 수정 - Orders Kafka Topic
___
- 사용자의 정보가 들어왔을 때 H2 라는 오더 서비스 DB에 저장 시켜 줬다. 
- DB에 저장 시키는 부분을 사용하지 않고 kafka에 전달 해 준다.
- Order Service의 Producer에서 발생하기 위한 메시지 등록

<br>

### 9. Orders Microservice 수정 - Order Kafka Producer
___
- orderService 주문하는 Controller 를 기존의 createOrder를 이용하여 h2 DB에 저장하는 로직 -> topic 을 보내는 것으로 변경

- 이 값은 임의로 지정하는 것이 아니라 kafka connector 를 통해 소스 커텍터에서 싱크 커넥터로 데이터를 보낼때 어떤 정보가 저장 될 것인지 필드를 구성해야 한다.
- Field.java : 데이터를 저장하기 위해 어떤 필드가 사용될 것인지 지정
```java
@Data
@AllArgsConstructor
public class Field { 
    private String type;
    private boolean optional;
    private String field;
}
```
```java
@Data
@Builder //
public class Payload {  //mariaDB의 컬럼과 동일
    private String order_id;
    private String user_id;
    private String product_id;
    private int qty;
    private int unit_price;
    private int total_price;
}
```
```java
@Data
@Builder
public class Schema {
    private String type;
    private List<Field> fields;
    private boolean optional;
    private String name;
}
```
```java
@Data
@AllArgsConstructor
public class KafkaOrderDto implements Serializable {
    private Schema schema;
    private Payload payload;
}
```
- 위에서 4개의 클래스로 만든 DTO 부분으로 이 코드를 사용할 수 있는 message class 생성
- producer 에서 발생하기 위한 메시지 생성 후 json 형태로 변경하여 send
```java
@Service
@Slf4j
public class OrderProducer {

    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    public OrderProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    List<Field> fields = Arrays.asList(new Field("sY" +
                    "tring", true, "order_id"),
            new Field("string", true, "user_id"),
            new Field("string", true, "product_id"),
            new Field("int32", true, "qty"),
            new Field("int32", true, "unit_price"),
            new Field("int32", true, "total_price")
            );

    Schema schema = Schema.builder()
            .type("struct")
            .fields(fields)
            .optional(false)
            .name("orders")
            .build();

    public OrderDto send(String topic, OrderDto orderDto) { // Producer에서 발생하기 위한 메시지 포멧으로 변경
        Payload payload = Payload.builder()
                .order_id(orderDto.getOrderId())
                .user_id(orderDto.getUserId())
                .product_id(orderDto.getProductId())
                .qty(orderDto.getQty())
                .unit_price(orderDto.getUnitPrice())
                .total_price(orderDto.getTotalPrice())
                .build();

        KafkaOrderDto kafkaOrderDto = new KafkaOrderDto(schema, payload);   // 위에서 메시지 생성
        
        ObjectMapper mapper = new ObjectMapper();
        String jsonInString = "";
        try {
            jsonInString = mapper.writeValueAsString(kafkaOrderDto);
        } catch (JsonProcessingException exception) {
            exception.printStackTrace();
        }

        kafkaTemplate.send(topic, jsonInString);
        log.info("Order Producer sent data from the Order microService : " + kafkaOrderDto);

        return orderDto;
    }

}
```
- jpa로 h2 DB에 저장 -> topic 보내는것 으로 변경
```java
@RestController
@RequestMapping("/order-service")
public class OrderController {
    Environment env;
    OrderService orderService;
    KafkaProducer kafkaProducer;
    OrderProducer orderProducer;
    
    @Autowired
    public OrderController(Environment env, OrderService orderService,
                           KafkaProducer kafkaProducer, OrderProducer orderProducer) {
        this.env = env;
        this.orderService = orderService;
        this.kafkaProducer = kafkaProducer;
        this.orderProducer = orderProducer;
    }
    
    @PostMapping("/{userId}/orders")
    public ResponseEntity<ResponseOrder> createOrder(@PathVariable("userId") String userId,
                                                     @RequestBody RequestOrder orderDetails) {
        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);

        OrderDto orderDto = mapper.map(orderDetails, OrderDto.class);
        orderDto.setUserId(userId);
        /* jpa */
//        OrderDto createOrder = orderService.createOrder(orderDto);
//        ResponseOrder responseOrder = mapper.map(createOrder, ResponseOrder.class);

        /* kafka */
        orderDto.setOrderId(UUID.randomUUID().toString());
        orderDto.setTotalPrice(orderDetails.getQty() * orderDetails.getUnitPrice());

        /* send this order to the kafka */
        kafkaProducer.send("example-catalog-topic", orderDto);
        orderProducer.send("orders", orderDto);

        ResponseOrder responseOrder = mapper.map(orderDto, ResponseOrder.class);

        return ResponseEntity.status(HttpStatus.CREATED).body(responseOrder);
    }

}
```

- zookeeper, kafka, kafka-connect 기동
- orders sink connector 등록 : POST 127.0.0.1:8083/connectors
```json
{
    "name" : "my-order-sink-connect",
    "config" : {
        "connector.class" : "io.confluent.connect.jdbc.JdbcSinkConnector",
        "connection.url":"jdbc:mariadb://localhost:3306/mydb",
        "connection.user":"root",
        "connection.password":"1234",
        "auto.create":"true",
        "auto.evolve":"true",
        "delete.enabled":"false",
        "task.max":"1",
        "topics":"orders"
    }
}
```
- 여러개의 오더 서비스에서 발생했던 메시지가 토픽에 잘 전달되고, 전달된 메시지가 하나의 단일 DB에 들어가야한다.

<br>

### 10. Orders Microservice 수정 - Order Kafka Producer
___
- OrderService 2개 기동
- 주문 : POST 127.0.0.1:8000/order-service/b785de1e-5b78-4ab9-a3aa-9f226830c129/orders
- 
```json
{
    "productId": "CATALOG-003",
    "qty": 12,
    "unitPrice": 1500
}
```
```text
Order Producer sent data from the Order microService : 
KafkaOrderDto(
schema=Schema(
type=struct, 
fields=
[Field(type=String, optional=true, field=order_id), 
Field(type=String, optional=true, field=user_id), 
Field(type=String, optional=true, field=product_id), 
Field(type=int32, optional=true, field=qty), 
Field(type=int32, optional=true, field=unit_price), 
Field(type=int32, optional=true, field=total_price)], 
optional=false, name=orders), 
payload=Payload(
order_id=ffcecf8a-63d2-49a6-a373-1c5a4f21e7a7, 
user_id=b785de1e-5b78-4ab9-a3aa-9f226830c129, 
product_id=CATALOG-003, qty=12, unit_price=1500, total_price=18000))
```
- .\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic orders --from-beginning 토픽에서도 확인가능
- 여러번 주문을 하면 order Service 1, 2번중 랜덤하게 로그가 찍힌다.
- mariadb 의 orders 테이블에 주문했던 데이터들이 1번이든 2번이든 같은 테이블에 쌓인 것을 확인 할 수 있다.

- CQRS 패턴 : DB에 저장되는 메시징 큐잉 시스템을 이벤트 소싱이라고 해서 데이터가 저장하는 파트, 데이터를 읽어오는 파트를 분리해서 만드는 것
