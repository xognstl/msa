## Docker Essentials - Image

### 1. Docker Image
___
- Docker Image(docker 실행 환경) 
  - 컨테이너를 만드는데 필요한 읽기 전용의 상태의 템플릿
  - 컨테이너 실행에 필요한 파일과 설정 값 등을 포함하고 있지만 상태 값은 X

- Docker Image 생성 방법
- Dockerfile : 이미지 빌드용 DSL
  - $ touch Dockerfile
  - $ vi Dockerfile
  ```dockerfile
  FROM ubuntu:16.04
  ```
  - $ docker build --tag first-image:0.1 -f Dockerfile .
  - $ docker image ls


- Dockerfile 작성 명령어 
  - FROM : Base Image 지정 명령어
  - RUN : 특정 Layer 생성
  - COPY : 이미지 파일 생성시 호스트 파일 복사
  - ADD : 이미지 파일 생성시 호스트 파일 복사(tar, url가능)
  - WORKDIR : 이미지 파일 생성시 명령어가 실행 될 작업 디렉토리
  - ENTRYPOINT : 컨테이너가 실행 될 때 가장 먼저 실행될 프로그램 지정(컨테이너 실행시 명령어 Overwrite 불가능)
  - CMD : 컨테이너가 실행될때 가장 먼저 실행 될 프로그램 지정(컨테이너 실행시 명령어 Overwrite 가능)
  - ENV : 컨테이너 내의 환경 변수 설정
  - EXPOSE : 컨테이너의 특정 포트를 외부에 오픈

<br>

### 2. Docker 를 이용하여 가상화 시스템 구축
___
- Docker 환경에서 웹 서비스 구축 하기(Node.js)
- alpine 만 쓰면 node 가 설치 안되있어 오류 -> node 가 설치 되어있는 alpine으로 변경
- packge.json 파일이 있어야 npm 명령어 실행가능 -> COPY 추가
- workdir 을 추가하면 copy 한 파일이 해당경로로 간다.
```dockerfile
FROM node:alpine
WORKDIR /home/app
COPY ./package.json ./package.json
COPY ./app.js ./app.js
RUN npm install
CMD ["node","app.js"]
```
- $ docker build -t nodejs-demo1:latest -f Dockerfile .
- $ docker run nodejs-demo1:latest
- $ docker run -p 8000:8000 --name my-node nodejs-demo1:0.2

<br>

### 3. Docker Registry
___
- Docker Image 저장소란? 
  - Docker를 통해 생성하는 Image들을 저장해주는 저장소 (Repository)
  - Docker Image들의 위치 제어 및 CI/CD를 위한 자동화 Pipeline 구축 가능
  - docker hub 에 push , pull 로 업로드 , 다운로드


- Local Registry 사용
  - $ docker run -d -p 5000:5000 --restart always --name registry registry:2 : local Registry 생성
    - http://127.0.0.1:5000/v2/_catalog 에서 등록되있는거 확인 할 수 있다.
  - $ docker tag nodejs-demo1:0.2 localhost:5000/nodejs-demo1:0.2
  - $ docker push localhost:5000/nodejs-demo1:0.2
    - http://127.0.0.1:5000/v2/nodejs-demo1/tags/list
  - $ docker tag nodejs-demo1:latest localhost:5000/nodejs-demo1:latest


- Cloud Registry 사용
  - docker hub 사이트에서 xognstl/nodejs-demo1 의 레포지토리 생성
  - $ docker tag nodejs-demo1:0.2 xognstl/nodejs-demo1:0.2
  - $ docker push xognstl/nodejs-demo1:0.2
