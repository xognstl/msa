## 섹션 13. 데이터 동기화를 위한 Apache Kafka의 활용 (2)

### 1. Orders Microservice와 Catalogs Microservice에 Kafka Topic의 적용
___
- Order Service -> Catalog Service 에 데이터를 Kafka 사용(상품 수량 업데이트)
- 현재 microService 의 DB는 각각 독립적으로 구성 해놓음
- Order Service 에 주문 정보, 주문했던 상품 수량 정보가 들어가고 Catalog Service 의 재고 수량도 - 해줘야한다. (동기화 필요)
- Order Service 에서 Kafka Topic 으로 메시지 전송(Producer)
- Catalog Service 에서 Kafka Topic 에 전송 된 메시지 취득 (Consumer)

- kafka dependency 등록

#### Catalog Service
- 빈등록
  - ConsumerFactory : 토픽에 접속하기 위한 접속 정보
  - ConcurrentKafkaListenerContainerFactory : 토픽에 어떤 변경 사항이 있는지 리스닝하는 리스너를 등록

- KafkaConsumer class 
  - 토픽에 변경된 데이터 값을 실제 DB에 반영 해주는 작업
  - @KafkaListener 를 가지고 있는 메소드에서 값이 변경되면 topic 에서 캐치해서 수량 update 해주는 로직을 작성

#### Order Service
- 빈등록
  - ProducerFactory 
  - kafka template : topic 에 데이터를 보내기 위해서 사용되는 객체

- KafkaProducer class
  - kafka template 가져와서 send 라는 동작 실행
  - 주문을 json 형태로 변경하여 kafka로 전달 하는 로직 작성

- 주문하는 Controller 수정
  - Kafka message 전달하는 코드 추가
