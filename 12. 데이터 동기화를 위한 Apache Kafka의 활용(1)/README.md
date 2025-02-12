## 섹션 12. 데이터 동기화를 위한 Apache Kafka의 활용 (1)

### 1. Apache Kafka 개요
___
- Apache Kafka : Apache Software Foundation 의 Scalar 언어로 된 오픈 소스 메시지 브로커 프로젝트
- 메시지 브로커 : 특정한 리소스에서 다른쪽에 있는 리소스, 서비스 시스템으로 메시지를 전달할 때 사용되는 서버
- 실시간으로 데이터 피드를 관리하기 위해 높은 처리량을 제공, 낮은 지연시간을 가지고 있는 플랫폼 사용가능
- 모든 시스템으로 데이터를 실시간으로 전송하여 처리할 수 있는 시스템
- 데이터가 많아지더라도 확장이 용이한 시스템
- Producer(메시지 보내는쪽)/ Consumer(받는쪽) 분리, 메시지를 여러 Consumer 에게 허용
- 높은 처리량을 위한 메시지 최적화, Scale-out 가능 

#### Kafka Broker
- 실행된 Kafka 애플리케이션 서버
- 3대 이상의 Broker Cluster 구성
- 어플리케이션 서버의 상태, 서버의 리더, 장애에 대한 체크, 장애 복구 관리할 수있는 코디네이터 시스템 Apache ZooKeeper 사용
- n개 Broker 중 1대는 Controller 기능 수행(각 broker 에게 담당 파티션 할당, broker 정상 동작 모니터링 관리)

<br>

### 2. Apache Kafka 설치
___
- https://kafka.apache.org/ => Scala 2.13  - kafka_2.13-3.9.0.tgz 다운
- tar xvf kafka_2.13-3.9.0.tgz 압축해제
- config : zookeeper.properties 주키퍼 실행 설정 파일, server.properties 카프카 서버 기동 설정 파일
- zookeeper-server-start.bat, kafka-server-start.bat 주키퍼 , 카프카 서버 실행 파일

<br>

### 3. Apache Kafka 사용 - Producer/Consumer
___
- 카프카와 데이터를 주고 받기 위해 사용하는 Java Library Kafka-clients
- Producer, Consumer, Admin, Stream 등 Kafka 관련 API 제공
- 다양한 3rd party library 존재 : https://cwiki.apache.org/confluence/display/KAFKA/Clients 확인 가능


#### 구동 방법
- Zookeeper, Kafka 서버 구동
  - $KAFKA_HOME/bin/zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties
  - .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
  - $KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties 
  - .\bin\windows\kafka-server-start.bat .\config\server.properties
- Topic 생성 / 프로듀서, 컨슈머를 콘솔 상태에서 테스트 할 수 있는 프로그램
  - 카프카는 프로듀서에서 메시지를 보내면 데이터는 Topic 이란 곳에 저장된다. 사용자가 임의로 토픽 생성 가능
  - $KAFKA_HOME/bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092 --partitions 1
  - .\bin\windows\kafka-topics.bat  --create --topic quickstart-events --bootstrap-server localhost:9092 --partitions 1
  - Topic에 데이터를 보내고, 토픽에 쌓였던 데이터가 다시 가져가는 방식
- Topic 목록 확인
  - $KAFKA_HOME/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  - .\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --list
- Topic 정보 확인
  - $KAFKA_HOME/bin/kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092
  - .\bin\windows\kafka-topics.bat --describe --topic quickstart-events --bootstrap-server localhost:9092
- 메시지 생산
  - $KAFKA_HOME/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic quickstart-events
  - .\bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic quickstart-events
- 메시지 소비
  - $KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic quickstart-events --from-beginning
  - .\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic quickstart-events --from-beginning
    
- cmd 창 4개 기동(zookeeper, kafka, producer, consumer)
- 주키퍼 실행 -> 카프카 실행 -> 토픽 생성 -> Producer 실행(메시지 생산) -> Consumer 실행(메시지 소비)
- Producer 에서 입력 하면 Consumer 에 입력한 메시지가 뜬다.
- 프로듀서가 해주는 역할은 컨슈머에 다이렉트로 데이터를 보내는 것이 아니라 컨슈머가 토픽에 관심이 있다고  
등록을 하게 되면 등록되어진 메시지를 토픽으로 부터 받는 역할을 하는게 컨슈머
- 토픽에다가 메시지를 전달하는게 프로듀서
- 컨슈머 하나 더키면 그전에 보냈던 메시지가 바로 뜨고 메시지를 또보내면 2개의 컨슈머에서 다 메시지가 나옴

<br>

### 4. Apache Kafka 사용 - Kafka Connect
___
- 데이터를 자유롭게 export, import 할 수 있는 기능
- 코드 없이 Configuration 으로 데이터 이동
- Standalone mode, Distribution mode 지원
  - restful api 통해 지원 
  - stream 또는 batch 형태로 데이터 전송 가능
  - 커스텀 connector 를 통한 다양한 plugin 제공(file, mysql, s3, hive etc..)

- source system(hive, jdbc) -> kafka connect source -> kafka cluster -> kafka connect sink -> target system(s3..)
- 저장되어 있는 자료를 복제잡업 , mysql 데이터 -> oracle 전달 할때 사용
- db -> db
- maria db 다운로드 ( 10.5.8 받음)
- db 초기화 .\bin\mariadb-install-db.exe --datadir=C:\mariadb-10.5.8-winx64\data  --service=mariaDB --port=3306 --password=1234

<br>

### 5. Orders Microservice에서 MariaDB 연동
___
- net start mariaDB , mysql.server start(linux)
- mysql -u root -p
- create database mydb; => db 생성 => use mydb; => db 사용
- mysql dependency 추가 -> h2 콘솔에서 접속가능 
```text
Driver Class : org.mariadb.jdbc.Driver
JDBC URL : jdbc:mysql://localhost:3306/mydb
root/1234

create table users(
id int auto_increment primary key,
user_id varchar(20),
pwd varchar(20),
name varchar(20),
created_at datetime default NOW()
);
```
- Kafka Connect 를 통해 데이터가 insert 데이터를 감지했다가 새로운 데이터베이스에 옮기는 작업 

<br>

### 6. Kafka Connect 설치
___
- Kafka Connect는 Apache Kafka에서 데이터를 쉽게 수집하고 전달할 수 있도록 설계된 데이터 통합 프레임워크
- Source Connector: 외부 시스템에서 데이터를 가져와 Kafka 토픽으로 보냄.
- Sink Connector: Kafka 토픽 데이터를 외부 시스템으로 보냄


- 설치 방법
- curl -O http://packages.confluent.io/archive/6.1/confluent-community-6.1.0.tar.gz
- tar xvf confluent-community-6.1.0.tar.gz
- Kafka connect 설정 
  - $KAFKA_HOME/config/connect-distributed.properties 사용 해도되고 ,  
  $KAFKA_CONNECT_HOME/etc/kafka/connect-distributed.properties 사용 해도됨.
- Kafka connect 실행
  - ./bin/connect-distributed ./etc/kafka/connect-distributed.properties
  - .\bin\windows\connect-distributed.bat .\etc\kafka\connect-distributed.properties

- Topic 확인 : 기존에 등록했던 Topic 이외에 connect를 기동하면 자동적으로 4개 추가로 생성   
.\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --list  
kafka connect 가 소스에서 읽어왔던 데이터들을 저장하기위한 토픽

```text
config\tools-log4j.properties 가 없다는 에러나면 
connect-distributed.bat의 log4j 경로를 
%BASE_DIR%/config/connect-log4j.properties에서 
%BASE_DIR%/etc/kafka/connect-log4j.properties로 바꾸어주면 해결 가능
```

- jdbc connector 다운(Source, Sink에서 사용) : https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc
- F:\msa_project\kafka-connect\confluent-7.3.1\etc\kafka\connect-distributed.properties 파일에 plugin.path 등록
- plugin.path=\F:\msa_project\kafka-connect\confluentinc-kafka-connect-jdbc-10.6.3\lib

- JDBC Connector 테스트를 위한 Mariadb JDBC Driver 복사
  - C:\Users\diquest\.m2\repository\org\mariadb\jdbc\mariadb-java-client\2.7.2 에 있는 JDBC Driver를 
  - F:\msa_project\kafka-connect\confluent-7.3.1\share\java\kafka 로 복사

<br>

### 7. Kafka Source Connect 사용
___
- Kafka Source Connect 추가
-  Kafka Source Connector 생성 : POST http://127.0.0.1:8083/connectors
```json
{
    "name" : "my-source-connect",
    "config" : {
        "connector.class" : "io.confluent.connect.jdbc.JdbcSourceConnector",
        "connection.url":"jdbc:mariadb://localhost:3306/mydb",
        "connection.user":"root",
        "connection.password":"1234",
        "mode": "incrementing",
        "incremental.column.name" : "id",
        "table.whitelist":"users",
        "topic.prefix" : "my_topic_",
        "tasks.max" : "1"
    }
}
```
- Kafka Connect 목록 확인
  - curl http://localhost:8083/connectors
- Kafka Connect 확인
  - curl http://localhost:8083/connectors/my-source-connect/status

- 테스트 방법
  - mariadb 에서 데이터 추가 =>  insert into users(user_id, pwd, name) values('user1', 'test1111', 'User name');
  - insert 후 토픽을 조회하면 아까 생성한 my-topic-users 를 조회할 수 있다.
  - .\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic my_topic_users --from-beginning
  - Connector 가 변경사항을 감지하면 토픽으로 이동시켜준다. 
  ```json
  {
    "schema":{
      "type":"struct",
      "fields":[
      {"type":"int32","optional":false,"field":"id"},
      {"type":"string","optional":true,"field":"user_id"},
      {"type":"string","optional":true,"field":"pwd"},
      {"type":"string","optional":true,"field":"name"},
      {"type":"int64","optional":true,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"created_at"}
      ],
      "optional":false,
      "name":"users"
    },
    "payload":{
      "id":2,
      "user_id":"admin",
      "pwd":"admin1",
      "name":"administrator",
      "created_at":1735312139000
    }
  }
  ```
  - 데이터베이스가 변동되면 토픽에 출력된다.

<br>

### 8. Kafka Sink Connect 사용
___
- Topic에 소스 커넥트에서 어떤 데이터를 전달하게되면 Topic에 그 데이터가 쌓인다.   
Sink connect는 토픽에 전달된 데이터를 가지고 와서 자기가 사용하는 것
- Users 테이블에 데이터가 insert, update 되면 해당 값이 Sink Connect 테이블에도 똑같이 반영된다.

-  Kafka Sink Connector 생성 : POST http://localhost:8083/connectors
```json
{
  "name" : "my-sink-connect",
  "config" : {
    "connector.class" : "io.confluent.connect.jdbc.JdbcSinkConnector",
    "connection.url":"jdbc:mariadb://localhost:3306/mydb",
    "connection.user":"root",
    "connection.password":"1234", 
    "auto.create":"true", //topic의 이름과 같은 테이블 생성 
    "auto.evolve":"true",
    "delete.enabled":"false",
    "task.max":"1",
    "topics":"my_topic_users"
  }
}
```
- show tables; 하면 my_topic_users 테이블 생성 확인 가능, topic에 있는 데이터도 복사된다.
-  users 테이블에 신규로 데이터를 insert 하면 my_topic_users 에도 똑같이 insert 되는것을 확인 할 수 있다.

#### Kafka Producer를 이용해서 Kafka Topic에 데이터 직접 전송
- Kafka Producer 데이터 전송 ->  Topic 에 추가 -> Maria DB 추가

- .\bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic my_topic_users : Producer 실행
- producer 에 직접 아래와 같이 포멧에 맞게 데이터 전달
```text
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":true,"field":"user_id"},{"type":"string","optional":true,"field":"pwd"},{"type":"string","optional":true,"field":"name"},{"type":"int64","optional":true,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"created_at"}],"optional":false,"name":"users"},"payload":{"id":4,"user_id":"xognstl","pwd":"user6","name":"user6","created_at":1735314642000}}
```
- users 테이블에는 금방 입력한 데이터는 없다. users는 토픽으로부터 데이터를 가져오는 것이 아니라 자신이 가지고 있는 데이터를  
토픽에 보내주는 역할을 하는 것이기 때문에 Producer가 집어넣은 데이터는 없다.
- 하지만 sink connect 와 연결되어 있는 my_topic_users 에는 데이터가 있다.
