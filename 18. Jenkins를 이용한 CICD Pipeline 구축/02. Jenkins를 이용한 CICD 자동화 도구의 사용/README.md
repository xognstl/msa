## Jenkins를 이용한 CI/CD 자동화 도구의 사용

### 1. CI/CD를 위한 Git, Maven 설정
___
- $docker run -d -v F:\jenkins:/var/jenkins_home -p 8088:8080 -p 50000:50000 --restart=on-failure --name jenkins-server -e HTTP_PROXY=http://10.10.10.213:3128 -e HTTPS_PROXY=http://10.10.10.213:3128 -e NO_PROXY=localhost,127.0.0.1 jenkins/jenkins
#### Git
- Manage Jenkins(Jenkins 관리) -> Jenkins Plugins(플러그인 관리) -> available -> github
- Manage Jenkins(Jenkins 관리) -> Global Tool Configuration -> git 설정해 주기
```text
git 사용가능 한지 확인
docker exec -it jenkins-server bash
jenkins@b57f9eaacbf7:/$ git --version
git version 2.39.5
```

#### Maven
- Manage Jenkins -> Jenkins Plugins -> available -> maven
- Manage Jenkins -> Global Tool Configuration -> maven

<br>

### 2. Maven 프로젝트 생성, Git에서 코드 가져와서 빌드하기
___
- item name : My-Second-Project (Maven Project)
- General : My maven project build
- Source Code Management 
  - None -> Git 
  - Repository URL : https://github.com/*****/cicd-web-project
  - Branch Specifier (blank for 'any') : */main
- Build
  - Root POM: pom.xml
  - Golds: clean compile package
- Save -> Build Now

- cd /var/jenkins_home/workspace 에 프로젝트 파일이 있는것을 확인 할 수 있다.

<br>

### 3. CI/CD 작업을 위한 Tomcat 서버 연동 / 배포
___
- Manage Jenkins -> Jenkins Plugins -> available -> deploy to container plugin
- 해당 플러그인이 jenkins에서 패키징 했던 war 파일을 tomcat 에 복사 할 수 있다.
- Item name -> My-Third-Project (Maven Project)
- General : Deploy the Second project on Tomcat
- Source Code Management
  - Repository URL : https://github.com/*****/cicd-web-project
- Build
  - Root POM: pom.xml
  - Golds: clean compile package
- Post-build Actions(빌드 후 조치)
  - Deploy war/ear to a container : **/*.war
  - Container: Tomcat 9.x Remote
  - Credentials: deployer(username/password)
  - Tomcat URL: http://172.10.72.88:8081/ : docker 에서 기동중이기때문에 127.0.0.1 안됨. 
- Save > Build Now
- tomcat의 webapp 에 hello-world.war 가 생긴걸 확인 할 수 있다.

#### tomcat 설정 변경
- C:\apache-tomcat-9.0.98\webapps\host-manager\META-INF\context.xml 에 아래내용 주석
- C:\apache-tomcat-9.0.98\webapps\manager\META-INF\context.xml 에도 주석
```xml
 <!-- <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />--> 
```
- C:\apache-tomcat-9.0.98\conf\tomcat-users.xml
```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="manager-jmx"/>
<role rolename="manager-status"/>
<user username="admin" password="admin" roles="manager-gui, manager-script, manager-jmx, manager-status"/>
<user username="deployer" password="deployer" roles="manager-script"/>
<user username="tomcat" password="tomcat" roles="manager-gui"/>
```

<br>

### 4. PollSCM 설정을 통한 지속적인 파일 업데이트
___
#### SSH Server 설치 
- $docker pull edowon0623/docker 
- $docker run --privileged --name docker-server -itd -p 10022:22 -p 8082:8080 -e container=docker -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cgroupns=host edowon0623/docker:latest /usr/sbin/init
- $ssh root@localhost -p 10022 /////P@ssw0rd
- $systemctl enable docker
- $systemctl start docker
- mobaXterm 같은거로 접속 할 수 있다.

#### PollSCM
- Project -> Configure -> Build Triggers
  - build periodically -> cron job (변경이 없어도 build 를 다시 한다.)
  - Poll SCM -> cron job (commit 내용을 감지하여 변경이 있을때만 다시 build)
  - 일정한 주기로 변경된 것을 감지하여 가져오는 작업

- Poll SCM 에 Schedule * * * * * : 매초마다 확인 

```text
* 깃에서 프로젝트 받아와서 내꺼에 올리기
git clone https://github.com/user/repository.git    // 소스 받아오기 
git remote add origin https://github.com/username/my-new-repo.git // 내깃 연결
git add .
git commit -m "message"
git push -u origin main
```

<br> 

### 5. SSH + Docker가 설치되어 있는 VM(컨테이너) 사용하기
___
- Manage Jenkins -> Jenkins Plugins -> available -> publish over ssh
- Manage Jenkins -> Configure System -> Publish over SSH
  - Add SSH Servers(SSH 서버의 정보 입력)
    - Name: docker-host
    - Hostname: [Remote IP] ex)192.168.0.8 => ssh 서버 ip 
    - Username: root
    - Passphrase/Password: P@ssw0rd
    - Port: 10022
  - Test Configuration -> 테스트 확인 버튼

<br> 

### 6. Docker Container에 배포하기
___
1. war 파일을 ssh를 이용해서 복사 (서버2)
2. (서버2) Dockerfile + war -> Docker Image 빌드
3. Docker Image -> 컨테이너 생성


- Item name : My-Docker-Project (Copy from: My-Third-Project)
- Build Triggers : Poll SCM -> Disable
- Post-build Actions
  - Deploy war/ear to a container(빌드 후 조치) -> Delete
  - 빌드 후 조치 추가 -> Send build artifacts over SSH
    - SSH Server
      - Name: docker-server(SSH에서 설정한 이름)
      - Transfer Set
        - Source files: target/*.war
        - Remove prefix: target
        - Remote directory: .  => ssh의 /root 에 war가 복사 된다.
- Saver -> Build Now

- root 에 war 파일 생긴걸 확인 할 수 있다.
```text
[root@7c1a55e661b3 ~]# cat Dockerfile
FROM tomcat:9.0
LABEL org.opencontainers.image.authors="xognstl@naver.com"
COPY ./hello-world.war /usr/local/tomcat/webapps
```
- $docker build -t docker-server -f Dockerfile . : 위의 Dockerfile 로 이미지 생성
- $docker run -p 8080:8080 --name mytomcat docker-server:latest

- localhost:8082 로 호출 
- docker server 의 컨테이너는 8082:8080 이고, SSH의 톰캣은 8080:8080
- http://localhost:8082/hello-world/ 를 해보면 정상적으로 호출된것을 확인할 수 있다.
