## 섹션 9. Spring Cloud Bus

### 1. Spring Cloud Bus
___

- 분산 시스템의 노드(microservice)를 경량 메시지 브로커(RabbitMQ)와 연결
- 상태 및 구성에 대한 변경 사항을 연결된 노드에게 전달(Broadcast)
- Spring cloud Config Server 에 각각 MicroService 에 연결이 되어있다. 각각의 MicroService가 가지고 있어야  
할 정보가 있다고 하면 Config Server와 연결되어있는 Spring Cloud Bus 가 MicroService 에 직접 데이터를 push 해준다.

<br>

- AMQP(Advanced Message Queuing Protocol) : 메시지 지향 미들웨어를 위한 개방형 표준 응용 계층 프로토콜
  - 메시지 지향, 큐잉, 라우팅(P2P, Publisher-Subcriber), 신뢰성, 보안
  - Erlang, RabbitMQ 에서 사용  

- Kafka 프로젝트 
  - Apache Software Foundation이 Scalar 언어로 개발한 오픈 소스 메시지 브로커 프로젝트
  - 분산형 스트리밍 플랫폼
  - 대용량의 데이터를 처리 가능한 메시징 시스템  

- Kafka는 대용량의 데이터를 빠른시간에 처리하고자 할때, RabbitMQ 는 적은 데이터를 안전하게 전달하는것을 보장

- RabbitMQ : 외부에서 별도의 클라이언트가 POST방식으로 busrefresh 라는 Actuator 호출(아무 서비스에 호출가능)

<br>

### 2. RabbitMQ 설치 - Windows 10
___

- erlang 설치 https://www.erlang.org/ => 환경변수 설정  
=>  C:\Program Files\Erlang OTP
- rabbitMQ 설치 https://www.rabbitmq.com/docs/download => 윈도우 서비스에 실행중 확인 가능, 환경변수 설정  
=> C:\Program Files\RabbitMQ Server\rabbitmq_server-4.0.5\sbin
- rqbbitMQ 플러그인 설치 powershell => rabbitmq-plugins enable rabbitmq_management
- http://127.0.0.1:15672/ 접속 guest/guest

<br>

### 3. AMQP 사용
___
- Config Server : AMQP for Spring cloud Bus, Actuator dependency 추가
- User, gateway Server : AMQP for Spring cloud Bus dependency 추가
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

- application.yml rabbitMQ 설정 추가
```yaml
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: guest
    password: guest
    
management:
  endpoints:
    web:
      exposure:
        include: health, busrefresh
```

- 테스트
* application.yml 파일 변경
* http://127.0.0.1:8888/config-service/default 변경확인 가능
* 127.0.0.1:8000/user-service/actuator/busrefresh => 변경 적용
- busrefresh 이벤트를 사용하여 메시지를 큐에 발행하면, 해당 이벤트를 구독하고 있는 서버들이 메시지를 가져가는 방식은 푸시 방식

