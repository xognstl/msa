## 섹션 17. 애플리케이션 배포 - Docker Container

### 1. 애플리케이션 배포 구성
___
#### Docker Network
- Bridge network
  - host PC 와 별도의 가상의 네트워크를 만들고 container 를 배치해 놓고 쓰는 방식
  - $docker network create --driver bridge [브릿지 이름]

- host network
  - host 네트워크와 게스트 OS 네트워크를 같은 환경에서 사용
  - 네트워크를 호스트로 설정하면 호스트의 네트워크 환경을 그대로 사용
  - 포트 포워딩 없이 내부 어플리케이션 사용

- None network
  - 네트워크를 사용하지 않음
  - io 네트워크만 사용, 외부와 단절

- $docker system prune : 중지되어 있는 컨테이너 , 불필요한 리소스 전부 삭제
- $docker network ls : 시스템의 네트워크 조회(처음에 조회하면 bridge, host, none 3개가 기본적으로 있다.)
- $docker network create --gateway 172.18.0.1 --subnet 172.18.0.0/16 ecommerce-network
  - 네트워크 생성
- $docker network inspect ecommerce-network : 네트워크 상세 보기
  - 할당된 서브넷, 게이트웨이 , 할당된 컨테이너등 확인 가능
- 컨테이너에서 사용하기 위한 네트워크를 직접 생성해서 사용하게 되면 일반적인 컨테이너는 하나의 게스트 OS  
각각의 게스트 OS 마다 고유한 IP 주소 할당, 컨테이너들 간에 IP 주소를 이용해 서로 통신을 한다.  
같은 네트워크에 포함된 컨테이너들끼리는 IP 주소 외에도 컨테이너 ID, 이름을 이용해 통신 가능  
ip address, id는 변경되도 name을 할당 할 수 있기때문에 변경되어도 name은 그대로기 때문에 그냥 사용가능

<br>

### 2. RabbitMQ
___
- hub.docker.com 에서 rabbitMQ 공식 TAG에서 적절한거 pull
```text
$docker run -d --name rabbitmq --network ecommerce-network
-p 15672:15672 -p 5672:5672 -p 15671:15671 -p 5671:5671 -p 4369:4369
-e RABBITMQ_DEFAULT_USER=guest
-e RABBITMQ_DEFAULT_PASS=guest rabbitmq:management

rqbbitmq:management : 실제로 호출하려는 이미지의 이름
```
- run 실행하면 이미지를 알아서 같이 다운받아주고 container 생성 진행
```text
CONTAINER ID   IMAGE                 COMMAND                   CREATED          STATUS          PORTS                                                                                                                        NAMES
39b410a7e69a   rabbitmq:management   "docker-entrypoint.s…"   34 seconds ago   Up 28 seconds   0.0.0.0:4369->4369/tcp, 0.0.0.0:5671-5672->5671-5672/tcp, 15691-15692/tcp, 0.0.0.0:15671-15672->15671-15672/tcp, 25672/tcp   rabbitmq
```
- $docker network inspect ecommerce-network
```text
"Containers": {
            "39b410a7e69a528a5108188dd3a93cf274cb3151ae32075074fc63819598cac5": {
                "Name": "rabbitmq",
                "EndpointID": "e9495023dcc470d49a95497ab220d41f5e506e985d510a3bd73b8b37857e0559",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
```
- http://127.0.0.1:15672/ 접속하면 rabbitMQ 접속 가능
- rabbitMQ는 호스트에서 꺼져있어도 15672번을 접속하게 되면 rabbitmq라는 컨테이너의 15672와 연결될 수 있게 포워딩 되어 있다.
- 재기동시 : docker run -d --name rabbitmq --network ecommerce-network -p 15672:15672 -p 5672:5672 -p 15671:15671 -p 5671:5671 -p 4369:4369 -e RABBITMQ_DEFAULT_USER=guest -e RABBITMQ_DEFAULT_PASS=guest rabbitmq:management

<br>

### 3. Configuration Service
___
- 기존 apiEncryptionKey.jks 를 config-service 프로젝트에 복사
- dockerfile 생성
```dockerfile
FROM openjdk:17-ea-11-jdk-slim
VOLUME /tmp
COPY apiEncryptionKey.jks apiEncryptionKey.jks
COPY target/config-service-1.0.jar ConfigService.jar
ENTRYPOINT ["java","-jar","ConfigService.jar"]
```
- keyFile 위치 변경
```yaml
encrypt:
  key-store:
#    location: file:///F:\msa_project\keystore\apiEncryptionKey.jks
    location: file:/apiEncryptionKey.jks
```
- $mvn clean compile package -DskipTests=true : 빌드 (테스트 코드 스킵)
- $docker build -t xognstl/config-service:1.0 . : docker 빌드
- $docker images : config-service 의 이미지 생성된 것 확인 가능

- application.yml 에 rabbitmq 정보가 있다. 127.0.0.1 은 host ip 주소고 docker container 에 할당된 ip로 변경 해야한다.
- 직접 yml 파일을 변경 해도 되지만 실행 할때 환경변수로 "spring.rabbitmq.host=rabbitmq" 변경 가능
- ip대신 이름으로 변경 가능

```text
$docker run -d -p 8888:8888 --network ecommerce-network \
-e "spring.rabbitmq.host=rabbitmq" \
-e "spring.profiles.active=default" \ 
--name config-service xognstl/config-service1.0

docker run -d -p 8888:8888 --network ecommerce-network -e "spring.rabbitmq.host=rabbitmq" -e "spring.profiles.active=default" --name config-service xognstl/config-service:1.0
```
- docker ps -a, docker network inspect ecommerce-network 추가된걸 확인할 수 있다.
- docker logs config-service log 도 확인

#### 다시 빌드
- mvn 업데이트 후 docker 빌드
- none 이라고 이미지가 생긴다.(기존꺼)
- $docker stop config-service => 프로세스 확인하면 exit 확인
- $docker system prune => 이미지 none 도 지워지고 stop container 도 없어짐
- proxy 도 추가 해줘야함. 
- $docker run -d -p 8888:8888 --network ecommerce-network -e "spring.rabbitmq.host=rabbitmq" -e "spring.profiles.active=default" -e "http_proxy=http://10.10.10.213:3128" -e "https_proxy=http://10.10.10.213:3128" --add-host http.docker.internal:10.10.10.213 --name config-service xognstl/config-service:1.0

```text
$docker exec -it config-service bash
- curl 다운
apt-get update
apt-get install -y curl
or 
FROM your-base-image
RUN apt-get update && apt-get install -y curl

$curl https://github.com/xognstl/spring-cloud-config
```
- proxy 추가랑 curl 확인으로 git yml 파일 가져오는것 해결

<br>

### 4. Discovery Service
___
```dockerfile
FROM openjdk:17-ea-11-jdk-slim
VOLUME /tmp
COPY target/discoveryservice-1.0.jar DiscoveryService.jar
ENTRYPOINT ["java","-jar","DiscoveryService.jar"]
```
- ecommerce 사용 할 수 있게 등록
```yaml
  cloud:
    config:
      uri: http://127.0.0.1:8888
      name: ecommerce
```

```text
$mvn clean compile package -DskipTests=true
$docker build --tag xognstl/discovery-service:1.0 .
$docker push xognstl/discovery-service:1.0
$docker run -d -p 8761:8761 --network ecommerce-network -e "spring.cloud.config.uri=http://config-service:8888" --name discovery-service xognstl/discovery-service:1.0
```

<br>

### 5. Apigateway Service
___
```dockerfile
FROM openjdk:17-ea-11-jdk-slim
VOLUME /tmp
COPY target/apigateway-service-1.0.jar ApiGateway.jar
ENTRYPOINT ["java","-jar","ApiGateway.jar"]
```
```text
$mvn clean compile package -DskipTests=true
$docker build --tag xognstl/apigateway-service:1.0 .
$docker push xognstl/apigateway-service:1.0
$docker run -d -p 8000:8000 --network ecommerce-network 
-e "spring.cloud.config.uri=http://config-service:8888"
-e "spring.rabbitmq.host=rabbitmq"
-e "eureka.client.serviceUrl.defaultZone=http://discovery-service:8761/eureka/"
 --name apigateway-service
  xognstl/apigateway-service:1.0
  
$docker run -d -p 8000:8000 --network ecommerce-network -e "spring.cloud.config.uri=http://config-service:8888" -e "spring.rabbitmq.host=rabbitmq" -e "eureka.client.serviceUrl.defaultZone=http://discovery-service:8761/eureka/" --name apigateway-service xognstl/apigateway-service:1.0
```
- Eureka Server 를 확인하면 잘 등록이 된 것을 확인 할 수 있다.

<br>

### 6. MariaDB
___
- 기존에 만들어 놨던 테이블 복사 예정
- C:\mariadb-10.5.8-winx64\data\mydb 위치에 보면 우리가 만들어 놨던 테이블 파일이 있다.
- C:\mariadb-10.5.8-winx64\data 폴더를 방금만든 F:\msa_project\docker-files\mysql_data 로 복사
- F:\msa_project\docker-files\mysql_data 경로에 dockerfile, data 

```dockerfile
FROM mariadb:10.7
ENV MYSQL_ROOT_PASSWORD 1234
ENV MYSQL_DATABASE mydb
COPY ./mysql_data/data /var/lib/mysql
EXPOSE 3306
CMD ["--user=root"]
```

```text
.\bin\mariadb-install-db.exe --datadir=C:\mariadb-10.5.8-winx64\data --service-mariaDB --port=3306 --password=1234
$docker build -t xognstl/my_mariadb:1.0 .
$docker run -d -p 3306:3306 --network ecommerce-network --name mariadb xognstl/my_mariadb:1.0 --innodb-force-recovery=1

## 접속 방법
F:\msa_project\docker-files>docker exec -it mariadb /bin/bash
root@ba4965d55fe2:/# mysql -h127.0.0.1 -uroot -p

- 어디서든 접속 할 수 있게 허용 
grant all privileges on *.* to 'root'@'%' identified by '1234'; 
flush privileges;
```
- $docker pull mariadb:10.7 버전 호환성 문제로 10.7 로 다운그레이드 먼저하고 진행하였다.

<br>

### 7. Kafka
___
- zookeeper + kafka standalone 
  - docker-compose 로 실행 : 도커 컨테이너를 하나의 스크립트 파일로서 실행할 수 있도록 해준다.  
  다른 부가적인 설정관련 된 것들을 모아놓고 명령어로 실행한다.
  - order service 에 kafka producer 사용할때 ip를 명시해주는데 네트워크에서 자동생성 ip는 안될 수 있으므로 ip지정
  - git clone https://github.com/wurstmeister/kafka-docker

- docker-compose-single-broker.yml 수정
```yaml
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    build: .
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 192.168.99.100
      KAFKA_CREATE_TOPICS: "test:1:1"
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
```yaml
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
    networks:
      my-network:
        ipv4_address: 172.18.0.100
  kafka:
    # build: .
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 172.18.0.101
      KAFKA_CREATE_TOPICS: "test:1:1"
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - zookeeper
    networks:
      my-network:
        ipv4_address: 172.18.0.101
networks:
  my-network:
    name: ecommerce-network
    external: true

```
- $docker-compose -f docker-compose-single-broker.yml up -d

<br>

### 8. Zipkin
___
- https://zipkin.io/pages/quickstart.html Docker 실행 방법
- $docker run -d -p 9411:9411 --network ecommerce-network --name zipkin openzipkin/zipkin

<br>

### 9. Monitoring
___  
- Prometheus
  - https://hub.docker.com/r/prom/prometheus

- yaml 파일 변경
```yaml
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: 'user-service'
    scrape_interval: 15s
    metrics_path: '/user-service/actuator/prometheus'
    static_configs:
    - targets: ['localhost:8000']
  - job_name: 'apigateway-service'
    scrape_interval: 15s
    metrics_path: '/actuator/prometheus'
    static_configs:
    - targets: ['localhost:8000']
  - job_name: 'order-service'
    scrape_interval: 15s
    metrics_path: '/order-service/actuator/prometheus'
    static_configs:
    - targets: ['localhost:8000'] 
```
```yaml
    static_configs:
      - targets: ["prometheus:9090"]
  - job_name: 'user-service'
    scrape_interval: 15s
    metrics_path: '/user-service/actuator/prometheus'
    static_configs:
    - targets: ['apigateway-service:8000']
  - job_name: 'apigateway-service'
    scrape_interval: 15s
    metrics_path: '/actuator/prometheus'
    static_configs:
    - targets: ['apigateway-service:8000']
  - job_name: 'order-service'
    scrape_interval: 15s
    metrics_path: '/order-service/actuator/prometheus'
    static_configs:
    - targets: ['apigateway-service:8000']  
```
```text
docker run -d -p 9090:9090 \
 --network ecommerce-network \
 --name prometheus \
 -v F:\msa_project\prometheus-3.1.0-rc.1.windows-amd64/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus 

$docker run -d -p 9090:9090 --network ecommerce-network --name prometheus -v F:\msa_project\prometheus-3.1.0-rc.1.windows-amd64/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

- Grafana
  - https://grafana.com/grafana/download?pg=get&plcmt=selfmanaged-box1-cta1
```text
docker run -d -p 3000:3000 \
 --network ecommerce-network \
 --name grafana \
 grafana/grafana

$docker run -d -p 3000:3000 --network ecommerce-network --name grafana grafana/grafana 
```

<br>

### 10. Deployed Services
___  
 
```dockerfile
FROM openjdk:17-ea-11-jdk-slim
VOLUME /tmp
COPY target/user-service-1.0.jar UserService.jar
ENTRYPOINT ["java","-jar","UserService.jar"]
```
```text
$mvn clean compile package -DskipTests=true
$docker build --tag xognstl/user-service:1.0 .
$docker push xognstl/user-service:1.0

docker run -d --network ecommerce-network
--name user-service
-e "spring.cloud.config.uri=http://config-service:8888" 
-e "spring.rabbitmq.host=rabbitmq" 
-e "spring.zipkin.base-url=http://zipkin:9411"
-e "eureka.client.serviceUrl.defaultZone=http://discovery-service:8761/eureka/" 
-e "logging.file=/api-logs/users-ws.log" # 로그파일 위치 지정 
xognstl/user-service:1.0

$docker run -d --network ecommerce-network --name user-service -e "spring.cloud.config.uri=http://config-service:8888" -e "spring.rabbitmq.host=rabbitmq" -e "spring.zipkin.base-url=http://zipkin:9411" -e "eureka.client.serviceUrl.defaultZone=http://discovery-service:8761/eureka/" -e "logging.file=/api-logs/users-ws.log" xognstl/user-service:1.0
```

```yaml
gateway:
  ip: 172.18.0.5
```
- $ docker logs -f user-service

<br>

### 11. Order Microservice
___  
```dockerfile
FROM openjdk:17-ea-11-jdk-slim
VOLUME /tmp
COPY target/order-service-1.0.jar OrderService.jar
ENTRYPOINT ["java","-jar","OrderService.jar"]
```
- kafaka 서버 ip주소 변경
```java
@EnableKafka
@Configuration
public class KafkaProducerConfig {
    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> properties = new HashMap<>();

        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "172.18.0.101:9092");
//        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9092");
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        return new DefaultKafkaProducerFactory<>(properties);
    }
}
```
```text
$mvn clean compile package -DskipTests=true
$docker build --tag xognstl/order-service:1.0 .
$docker push xognstl/order-service:1.0

docker run -d --network ecommerce-network
--name order-service
-e "spring.zipkin.base-url=http://zipkin:9411"
-e "eureka.client.serviceUrl.defaultZone=http://discovery-service:8761/eureka/"
-e "spring.datasource.url=jdbc:mariadb://mariadb:3306/mydb" 
-e "logging.file=/api-logs/order-ws.log" 
xognstl/order-service:1.0

$docker run -d --network ecommerce-network --name order-service -e "spring.zipkin.base-url=http://zipkin:9411" -e "eureka.client.serviceUrl.defaultZone=http://discovery-service:8761/eureka/" -e "spring.datasource.url=jdbc:mariadb://mariadb:3306/mydb" -e "logging.file=/api-logs/order-ws.log" xognstl/order-service:1.0
```

<br>

### 12. Catalog Microservice
___  
- KafkaConsumerConfig.java kafka ip 주소 변경
```dockerfile
FROM openjdk:17-ea-11-jdk-slim
VOLUME /tmp
COPY target/catalog-service-1.0.jar CatalogService.jar
ENTRYPOINT ["java","-jar","CatalogService.jar"]
```
```text
$mvn clean compile package -DskipTests=true
$docker build --tag xognstl/catalog-service:1.0 .
$docker push xognstl/catalog-service:1.0

docker run -d --network ecommerce-network
--name catalog-service
-e "eureka.client.serviceUrl.defaultZone=http://discovery-service:8761/eureka/"
-e "logging.file=/api-logs/catalog-ws.log" 
xognstl/catalog-service:1.0

$docker run -d --network ecommerce-network --name catalog-service -e "eureka.client.serviceUrl.defaultZone=http://discovery-service:8761/eureka/" -e "logging.file=/api-logs/catalog-ws.log" xognstl/catalog-service:1.0
```

<br>

### 13. Test
___  
- POST 127.0.0.1:8000/user-service/users
```json
{
    "email": "xognstl@naver.com",
    "name": "xognstl",
    "pwd": "test1234"
}
```
- POST 127.0.0.1:8000/order-service/사용자id/orders

<br>

### 14. Multi Profiles
___
- $mvn spring-boot:run -Dspring-boot.run.arguments=--spring.profiles.active=production
- $java -Dspring.profiles.active=default XXX.jar
