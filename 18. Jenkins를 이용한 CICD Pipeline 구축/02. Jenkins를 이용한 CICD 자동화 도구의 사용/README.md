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
- Manage Jenkins  Jenkins Plugins  available  maven
- Manage Jenkins  Global Tool Configuration  maven

<br>

### 2. Maven 프로젝트 생성, Git에서 코드 가져와서 빌드하기
___
- item name : My-Second-Project (Maven Project)
- General : My maven project build
- Source Code Management 
  - None -> Git 
  - Repository URL : https://github.com/joneconsulting/cicd-web-project
  - Branch Specifier (blank for 'any') : */main
- Build
  - Root POM: pom.xml
  - Golds: clean compile package
- Save -> Build Now

- cd /var/jenkins_home/workspace 에 프로젝트 파일이 있는것을 확인 할 수 있다.

<br>

### 3. CI/CD 작업을 위한 Tomcat 서버 연동 / 배포
___
- Manage Jenkins  Jenkins Plugins  available  deploy to container plugin
- 해당 플러그인이 jenkins에서 패키징 했던 war 파일을 tomcat 에 복사 할 수 있다.
- Item name  My-Third-Project (Maven Project)
- General : Deploy the Second project on Tomcat
- Source Code Management
  - Repository URL : https://github.com/joneconsulting/cicd-web-project
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
