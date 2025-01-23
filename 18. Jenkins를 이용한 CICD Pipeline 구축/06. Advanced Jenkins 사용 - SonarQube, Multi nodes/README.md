## Advanced Jenkins 사용 ② - SonarQube, Multi nodes

### 1. SonarQube 사용하기
___
- Continuous Integration + 분석 솔루션
    - 불필요한 리소스 낭비, 코드의 품질을 높혀주기위해 체크를 하고 리포트를 해주는 툴
    - Code Quality Assurance tool -> Issues + Defect + Code Complexity : 코드 분석도구
    - 버그 , 취약성 찾는 용도
    - 코드의 이상여부 탐지 
- $ docker pull sonarqube : 이미지 다운
- $ docker run --rm -p 9000:9000 --name sonarqube sonarqube : 컨테이너 실행
- admin/admin 으로 접속

<br>

### 2. SonarQube + Maven 프로젝트 사용하기
___
- Maven Project에 Sonarqube Plugin 추가, version 은 maven 사이트에서 확인
```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>5.0.0.4389</version>
</plugin>
```
- SonarQube token 생성 
  - My Account -> Security -> User token 생성
  - squ_4fbe84d9270fb8495bc99e01b577e3e6b04fa378 , 한번생성하면 다음에 못봄
- Maven build
  - $ mvn sonar:sonar -Dsonar.host.url=http://IP_address:9000 -Dsonar.login=[the-sonar-token]
  - $ mvn sonar:sonar -Dsonar.host.url=http://localhost:9000 -Dsonar.login=squ_4fbe84d9270fb8495bc99e01b577e3e6b04fa378
- SonarQube Projects 확인

#### BadCode 조사하기
- system out 코드 추가
- $ mvn clean compile package –DskipTests=true
- $ mvn sonar:sonar -Dsonar.host.url=http://IP:9000 -Dsonar.login=[the-sonar-token]

<br>

### 3. Jenkins + SonarQube 연동
___
- jenkins plugin 설치 : SonarQube Scanner 
- Manage Jenkins -> manage Credentials -> Add Credentials 에 token 등록(secret text)
- jenkins 관리 -> System -> SonarQube servers
  - Environment variables 선택
  - Name: SonarQube 서버의 이름지정
  - Server URL: SonarQube 서버의 IP address
  - Server authentication token: Credentials에서 지정한 Token 정보

- sonarqube 정보 pipeline script 에 추가(build 다음에 추가)
```text
stage('Sonarqube analysis') {
    steps {
       withSonarQubeEnv('SonarQube-server'){
           sh 'mvn sonar:sonar'
       }
    }
}
```

<br>

### 4. Jenkins Multi nodes - Master + Slaves
___
- Jenkins Master
  - 기본 Jenkins 서버 -> Master node
  - 빌드 작업 예약 및 빌드 실행
  - Slave에게 Build 할 Job 전달
  - Slave 모니터링
  - 빌드 결과 기록

- Jenkins Slave
  - Remote에서 실행되는 Jenkins 실행 Node
  - Jenkins Master의 요청 처리
  - Master로부터 전달된 Job 실행
  - 다양한 운영체제에서 실행 가능
  - Jenkins 프로젝트 생성 시 특정 Slave를 선택하여 실행 가능

<br>

### 5. Jenkins Node 추가하기
___
- 새로운 서버 추가
  - $ docker run --privileged --name jenkins-node1 -itd -p 30022:22 -e container=docker -v /sys/fs/cgroup:/sys/fs/cgroup --cgroupns=host edowon0623/docker:latest /usr/sbin/init
  - java 설치 : $ yum install -y java-11-openjdk-devel.aarch64
  - master -> node ssh 접속을 위한 키등록 
    - jenkins 서버 bash 로 접속
    - $ssh-keygen -> $ ssh-copy-id root@172.17.0.6
    - ssh root@172.17.0.6 으로 접속 해보면 비밀번호 필요없다.

- jenkins master 에 slave node 추가
- jenkins 관리 -> manage nodes -> new node
  - Node Name: slave1
  - Description: Add a server as a slave machine
  - Number of executors: 5 -> 작업 요청 처리 최대 요청 갯수
  - Remote root directory: /root/slave1 => 직접 생성 필요
  - Labels: slave1
  - Usage: Use this node as much as possible
  - Launch method: Launch agents via SSH
    - Host: [Slave server IP address] ex) 172.17.0.6
    - Port: 22
    - Credentials: root/P@ssw0rd , id  = slave1_root


- My-First-Project 수정
  - Restrict where this project can be run 선택
  - Label Expression : slave1

<br>

### 6. Jenkins Slave Node에서 빌드하기
___
- Node 2번 서버 생성
  - $docker run --privileged --name jenkins-node2 -itd -p 30023:22 -e container=docker -v /sys/fs/cgroup:/sys/fs/cgroup --cgroupns=host edowon0623/docker:latest /usr/sbin/init
  - java 설치 17버전
  - 키등록
  - master 에 slave node 추가

- script 에 agent node서버 추가해주면 해당 서버에서 할 수 있다.
```text
agent {
        label 'slave1'
    }
```
