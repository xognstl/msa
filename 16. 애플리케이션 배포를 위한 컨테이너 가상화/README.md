## 섹션 16. 애플리케이션 배포를 위한 컨테이너 가상화

### 1. 컨테이너 가상화
___
#### Virtualization
- 물리적인 컴퓨터 리소스를 다른 시스템이나 애플리케이션에서 사용할 수 있도록 제공
  - 플랫폼 가상화
  - 리소스 가상화

- 하이퍼바이저(Hypervisor)
  - Virtual Machine Manager (VMM)
  - 다수의 운영체제를 동시에 실행하기 위한 논리적 플랫폼
  - Type 1 : Native or Bare-metal (hardware - hypervisor - os)
  - Type 2 : Hosted (hardware - os - hypervisor - os)

#### Container Virtualization
- OS Virtualization
  - Host OS 위에 Guest OS 전체를 가상화
  - VMWare, VirtualBox
  - 자유도가 높으나, 시스템에 부하가 많고 느려짐

- Container Virtualization
  - Host OS가 가진 리소스를 적게 사용하며, 필요한 프로세스 실행
  - 최소한의 라이브러리와 도구만 포함
  - Container 의 생성속도 빠름

#### Container Image
- Container 실행에 필요한 설정 값
  - 상태값 x, Immutable

- Image를 가지고 실체화 -> Container
  - public(docker hub), private registry 에 이미지를 올린다.  


<img src="https://github.com/user-attachments/assets/1a68884a-cbd2-4151-8704-69ab8fa35511" width="700" height="500">  

#### DockerFile
- Docker Image 를 생성하기 위한 스크립트 파일
- 자체 DSL(Domain-Specific language) 언어 사용 -> 이미지 생성과정 기술

<br>

### 2. Docker 컨테이너
___
- https://www.docker.com/products/docker-desktop
- $docker info : 현재 설치된 도커의 정보
- $docker image ls : 현재 가지고 있는 이미지 목록
- $docker container ls : 현재 실행중인 컨테이너 목록
- $docker run [OPTIONS] IMAGE[:TAG|@DIGEST][COMMAND][ARG...]
  - -d : detached mode (백그라운드모드)
  - -p : 호스트(ex. window)와 컨테이너의 포트를 연결(포워딩)
  - -v : 호스트와 컨테이너의 디렉토리를 연결 (마운트)
  - -e : 컨테이너 내에서 사용할 환경변수 설정
  - --name : 컨테이너 이름 설정
  - --rm : 프로세스 종료시 컨테이너 자동 제거
  - -it : 터미널 입력을 위한 옵션
  - --link : 컨테이너 연결[컨테이너명:별칭]
  - ex) $docker run ubuntu:16.04

- tag 이름을 명시하지 않게되면 기본적으로 latest 이름이 자동으로 붙는다.(버전같은 느낌으로 사용) 
- https://hub.docker.com/ 에 ubuntu 검색 하면 ubuntu 관련 이미지 같은 정보를 볼 수 있다.
- $docker pull ubuntu:16.04 로 우분투 다운로드
- $docker run ubuntu:16.04 실행 하면 바로 꺼진다. 
- 우분투는 프로세스를 지속적으로 진행할 만한 코드가 없다. dockerfile에 프로세스를 지속적으로 운영하기 위한  
커맨드가 포함되어있지 않다. 우분투 자체 사용을 하는게 아니라 이미지가 베이스 이미지가 되어 추가적인 어플리케이션  
과 서비스 같은 것들을 만들때 사용된다.
- $docker container ls -a 종료된 이미지도 확인 가능
- 삭제 방법 : $docker container rm [컨테이너 ID] 

<br>

### 3. 컨테이너 생성과 실행
___
- docker 로 mysql 실행
- $docker run -d -p 3306:3306 -e MYSQL_ALLOW_EMPTY_PASSWORD=true --name mysql mysql:5.7
  - mysql:5.7 이미지의 이름 , 5.7은 루트 패스워드를 설정해주어야한다. 
  - 루트 패스워드를 지정하지 않기 위해서 -e MYSQL_ALLOW_EMPTY_PASSWORD 지정
- $docker exec -it mysql bash
  - exec : 추가적으로 컨테이너에 커맨드를 전달하고자 할때 사용
  - it : 명령어 전달

#### 다운 및 실행

- $docker pull mariadb:latest : mariadb 이미지 다운로드
- $docker run -d -p 3306:3306 -e MYSQL_ALLOW_EMPTY_PASSWORD=true --name mariadb mariadb:latest
  - 3306은 윈도우에서 사용중이므로 13306:3306 으로 실행
```text
$docker ps -a => UP 작동중 확인 가능
CONTAINER ID   IMAGE            COMMAND                   CREATED          STATUS          PORTS                     NAMES
38b04e91d5d5   mariadb:latest   "docker-entrypoint.s…"   46 seconds ago   Up 45 seconds   0.0.0.0:13306->3306/tcp   mariadb
```
- $docker logs (container ID or NAME)
- $docker exec -it mariadb /bin/bash
- root@38b04e91d5d5:/# => 루트아이디@컨테이너가 가지고 있는 id가 호스트 네임이 된다.
- $mysql -uroot -p -h127.0.0.1 로 로그인 했으나 mariadb 최신버전은 이게아니라 mariadb 라고 치면 로그인 된다.
- create database mydb; 로 mydb 라는 database 생성
- db tool 을 이용하여 127.0.0.1 : 13306 root 로 연결 확인
- 컨테이너 내부에서 접속도 가능
  - $docker stop mariadb : process exit 상태
  - 컨테이너를 삭제하면 컨테이너에 만들어놨던 mydb, log가 삭제된다. 그래서 컨테이너 외부에 저장하는 방법을 사용

<br>

### 4. Docker 이미지 생성과 Public registry에 Push
___
- $docker build -t xognstl/user-service:1.0 . 
  - dockerfile을 이미지로 만들때 build 사용 , .은 현재 위치
- $docker push xognstl/user-service:1.0
- $docker pull xognstl/user-service:1.0
  - hub에 push , pull 가능

- user-service 프로젝트 바로 밑에 dockerfile 생성
- hub 사이트에서 open jdk 검색 후 17-ea-jdk-slim TAG 사용
- $docker pull openjdk:17-ea-11-jdk-slim 
```dockerfile
FROM openjdk:17-ea-11-jdk-slim
VOLUME /tmp
COPY target/user-service-1.0.jar UserService.jar
ENTRYPOINT ["java","-jar","UserService.jar"]
```
- user-service mvn clean : mvn clean package -DskipTests=true
- 0.0.1-SNAPSHOT 이름 너무 길면 pom.xml 에서 수정
- $docker build --tag xognstl/user-service:1.0 .
- $docker push xognstl/user-service:1.0
- hub 사이트의 respository 에 정상 업로드 된걸 확인 할 수 있다.
- $docker rmi 393c4dda53d5 금방 생성한 이미지 삭제 
- $docker pull xognstl/user-service:1.0 => hub 에서 다시 받아옴
- $docker run xognstl/user-service:1.0 실행

