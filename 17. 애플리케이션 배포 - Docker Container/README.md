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
