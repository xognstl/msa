## Docker Essentials - Container

### 1. Docker Image
___
- 이미지 : docker 를 통해 실행하고자 하는 OS, 미들웨어, 서비스 어플리케이션 같은 서비스 형태가 제공대기 위한 하나의 템플릿
  - Layer 저장 방식 : 여러개의 layer 를 하나의 파일 시스템으로 사용
- Dockerfile : docker image 를 생성하기 위한 스크립트 파일
  - 자체 DSL 언어 사용 -> 이미지 생성과정 기술
  - 서버에 프로그램을 설치하는 과정을 dockerfile 로 기록 관리
  - 소스와 함께 버전관리가 되며, 누구나 수정 가능

<br>

### 2. Docker Container
___
- Container(인스턴스) : Docker Image 를 실행한 상태
  - 격리 된 시스템자원 및 네트워크를 사용할수 있는 독립적인 실행 단위
  - 읽기 전용 상태인이미지에 변경 된 사항을저장할수 있는 컨테이너 계층에저장
#### 명령어
- $docker run [OPTIONS]IMAGE[:TAG|@DIGEST] [COMMAND] [ARG…]
  - run : create , start, running 를 한번에 할 수 있다. 
  - -d, --detach : Detached mode, Background mode
  - -p, --publish : 호스트와 컨테이너의 포트를 연결(포워딩)
  - -v, --volume : 호스트와 컨테이너의 디렉토리를 연결(마운트)
  - -e, --env : 컨테이너내에서 사용할 환경변수 설정
  - --name : 컨테이너 이름 설정
  - --rm : 프로세스 종료시 컨테이너 자동 제거
  - -it : -i와 –t를 동시에 사용한것으로 터미널 입력을 위한 옵션
  - -link : 컨테이너연결[컨테이너명:별칭]


### 2. Docker Container
___
- 명령어
  - $ docker container ls [OPTIONS] ( = docker ps )
  - $ docker container stop [OPTIONS] CONTAINER [CONTAINER …]
  - $ docker container rm [OPTIONS] CONTAINER [CONTAINER …]
  - $ docker container logs ${CONTAINER_ID}
    - ex) docker logs -f $[CONTAINER_ID}
  - $ docker container exec [OPTIONS] CONTAINER COMMAND [ARG…]
    - ex) docker exec -it mysql /bin/bash
  - $ docker container inspect ${CONTAINER_ID} : 내용, 옵션, 환경 조건 조회
  - $ docker image ls [OPTIONS] [REPOSITORY[:TAG]] = docker images
  - $ docker image rm [OPTIONS] IMAGE [IMAGE …] 
  - $ docker image pull [OPTIONS] NAME[:TAG|@DIGEST]
  - $ docker system prune : 불필요한 이미지, 컨테이너 삭제 

#### Docker Container 실행 
- ubuntu container 생성
  - $ docker run ubuntu:16.04
  - $ docker run --rm -it ubuntu:16.04 /bin/bash


- MySQL 
  - $ docker run -d -p 3306:3306 -e MYSQL_ALLOW_EMPTY_PASSWORD=true --name mysql mysql:5.7
  - $ docker exec -it mysql bash


- Maria db
  - $ docker run -d -e MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=true --name my-mariadb mariadb
  - $ docker container exec -it my-mariadb bash
  - $ mariadb -h127.0.0.1 -uroot -p : mariadb 접속

<br>

### 3. 외부와의 Port Mapping
___
- 호스트 시스템에서 도커 컨테이너 Port 를 사용하기 위해서는 Port Mapping 필요
- $ docker run -p host_port:container_port <IMAGE NAME>


- nginx 
  - $ docker run nginx
  - $ docker exec -it 3eaea5eef46c bash
  - $ docker run -d -p 80:80 nginx : 호스트 pc 에서 접근 가능


- mariaDB
  - $ docker run -d -p 3306:3306 -e MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=true --name my-mariadb mariadb
  - dbeaver 같은 툴로 localhost:3306 으로 접근 가능


- Node.js
  - $ git clone https://github.com/joneconsulting/devops-docker.git : 프로젝트 다운
  - $ git checkout section1 : branch 변경
  - app.js, package.json 이 있는 폴더로 이동
  - $ docker run -it -v .:/app -p 8000:8000 node:alpine sh : 실행
  - $ npm install express : node js 실행시 필요한거 설치
  - $ node app.js : node.js 기동
  - http://localhost:8000/?name=xognstl : 호출 
