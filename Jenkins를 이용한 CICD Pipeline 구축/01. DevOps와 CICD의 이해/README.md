## DevOps와 CI/CD의 이해

### 1. Waterfall vs Agile
___
- Waterfall : 프로젝트의 각 단계가 구분이 뚜렷하게 나누어져 있는 순차적인 프로젝트 관리 방법론
- Agile : 프로젝트의 생명 주기 동안 반복적인 과정을 통해 점점 소프트웨어가 진화되는 과정

<br>

### 2. Cloud Native Application의 구성요소
___
- Cloud Native Application 의 구성 요소
  - 마이크로 서비스
  - 컨테이너 가상화 기술 : 클라우드 환경에 배포, 사용
  - DevOps : 서비스에 문제가 발생했을때 수정, 배포, 
    - 엔지니어가 프로그래밍하고, 빌드하고, 직접 시스템에 배포 및 서비스를 RUN  
    - 사용자와 끊임 없이 Interaction 하면서 서비스를 개선해 나가는 일련의 과정
  - CI/CD : 자동화 파이프라인에 의해 통합, 빌드, 테스트, 배포

<br>

### 3. CI/CD 자동화 도구의 이해, Work flow
___
- 지속적 통합(Continuous Integration)과 지속적 제공/배포(Continuous Delivery/Deployment)
- 개발된 결과물에 대해 지속적인 통합과 지속적인 배포 프로세스
- 개발된 어플리케이션을 통합하고 빌드, 테스트, 배포에 대한 자동화 처리
- CI 작업에는 컴파일, 테스트, 패키징, CD 작업은 패키지된 결과물을 서버에 배포 하는 작업

#### Work Flow
- git 
- CI 도구 : Jenkins(자동화 파이프라인 구축, 소스코드의 빌드, 테스트, 패키징 작업), build : Maven , Gradle
- IAC(Infrastructure as Code) : 서버의 인프라 스트럭처 관리 , Ansible, Terraform
- 배포 : Docker , Kubernetes
- aws, google CloudPlatform, Naver Cloud 

<br>

### 4. Jenkins
___
- CI/CD 파이프라인, Work Flow 를 설계하는데 사용하는 도구
- 다양한 Plugins 사용
  - build(maven, gradle), VCS(git,svn), laguages(java, python, node.js)

#### 다운로드
- https://www.jenkins.io/download/
- $docker pull jenkins/jenkins:lts-jdk11 
```text
$docker  run -d -v E:\jenkins:/var/jenkins_home -p 8088:8080 -p 50000:50000 --restart=on-failure --name jenkins-server jenkins/jenkins
```
- 다운이 완료되면 logs 에 f4b8432498114a5a8d29f624655d4e4f 같이 초기 비밀번호 확인가능
- http://127.0.0.1:8088/ 접속하면 password 입력 => Install suggested plugins 
- 계정 생성 => dashboard 접속
- jenkins 관리 -> Global Tool Configuration

<br>

### 5. 첫번째 Item(Project) 생성
___
- 새로운 item -> Freestyle project  
- Build > Execute shell
```shell
echo "welcome to my first project using Jenkins"
java -version
```
- 저장 후 dashboard 에 방금 생성한 item 클릭 -> build now 선택 -> Console output 확인 
```text
Started by user 강태훈
Running as SYSTEM
Building in workspace /var/jenkins_home/workspace/My-First-Proejct
[My-First-Proejct] $ /bin/sh -xe /tmp/jenkins5810953224925782940.sh
+ echo welcome to my first project using Jenkins
welcome to my first project using Jenkins
+ java -version
openjdk version "17.0.13" 2024-10-15
OpenJDK Runtime Environment Temurin-17.0.13+11 (build 17.0.13+11)
OpenJDK 64-Bit Server VM Temurin-17.0.13+11 (build 17.0.13+11, mixed mode)
Finished: SUCCESS
```

#### WorkSpace 확인
- $docker exec -it jenkins-server bash
- /var/jenkins_home/workspace/My-First-Proejct 
