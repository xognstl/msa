## Building and Managing Containerized Application

### 1. Docker Compose
___
- Docker compose : 컨테이너들을 관리하기 위한 기술
  - Container 애플리케이션을 정의하고 실행하는 도구
  - 한 번에 여러 개의 컨테이너 동시 실행 -> 각 컨테이너 별로 별도의 명령어 실행 가능
  - 다른 컨테이너와의 접속을 쉽게 구성
    - Docker 생성, 설정 관련된 작업을 작성해 놓은 script 파일 사용(docker compose 파일)


- Docker Compose 파일 작성
  - docker-compose.yml 형태로 작성
  - $ docker-compose up (파일명 없으면 docker-compose.yml)
  - $ docker-compose -f docker-compose1.yml up
  - $ docker-compose -f docker-compose1.yml up 이름1, 이름2 => 원하는 컨테이너만 따로 실행시킬 수 있다.


- docker-compose -f docker-compose1.yml up
```yaml
version: "3.1"
services:
  my-webserver:
    platform: linux/amd64 # for apple chip
    image: nginx:latest
    ports:
      - 80:80
```
- 실행 시 nginx 접속 가능 확인
- $ docker-compose -f docker-compose1.yml ps : 현재 docker compose 에서 작동 중인 상태 확인
- $ docker-compose -f docker-compose1.yml down : compose 종료 


- docker-compose.yml 파일 작성 명령어
  - 여러개의 컨테이너 기동시 병렬로 시작이되어 어떤게 먼저 기동될지 모른다. 그래서 연관관계가 있으면 에러가 날수도 있다. =>depends_on
  - image : Docker Image 지정
  - build : Dockerfile을 이용하여 Docker Image 빌드
  - command/entrypoint : 컨테이너안에서 작동하는 명령어 지정
  - ports/expose : 컨테이너 간 통신을 위한 Port 설정
  - depends_on : 서비스 의존 관계 정의(절대적X)
  - environment/env_file : 컨테이너 환경변수 지정
  - container_name/labels : 컨테이너 정보 설정
  - volumes/volumes_from : 컨테이너 데이터 관리

- Docker Compose 실행 명령어 =>  $ docker-compose <OPTIONS>
  - up : 컨테이너 생성/시작
  - ps : 컨테이너 목록표시
  - logs : 컨테이너 로그출력
  - run : 컨테이너 실행
  - stop : 컨테이너 정지
  - port : 공개 포트 번호 표시
  - rm : 컨테이너 삭제
  - config : 구성확인
  - down 컨테이너/리소스삭제

<br>

### 2. Docker Compose를 활용한 IT 서비스 구축
___
- DB, back-end, front-end 를 docker compose 를 이용하여 구동
- $ docker network create my-network : compose 에 network 에 my-network 가 없으면 에러가 난다.
- $ docker-compose -f docker-compose4.yml up -d : 실행
- $ docker-compose -f docker-compose4.yml logs
- http://localhost/ 에서 front-end 확인 가능
```yaml
version: "3.1"
services:
  my-db:
    platform: linux/amd64
    image: mariadb:latest
    environment: 
      - MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=true
      - MARIADB_DATABASE=mydb
    ports:
      - 13306:3306
    volumes: 
      - C:\xognstl\docker\docker-compose\data:/var/lib/mysql 
    networks:
      - my-network

  my-backend:
    image: edowon0623/my-backend:1.0
    environment: 
      - spring.datasource.url=jdbc:mariadb://my-db:3306/mydb
      - spring.datasource.password=
    ports:
      - 8088:8088
    depends_on:
      - my-db
    networks:
      - my-network

  my-frontend:
    image: edowon0623/my-frontend:1.0
    ports:
      - 80:80
    depends_on:
      - my-db
    networks:
      - my-network

networks:
  my-network:
    external: true
```
