## AWS 클라우드 환경에 배포하기

### 1. AWS EC2 인스턴스 생성
___
- AWS 홈페이지 접속하여 EC2 -> 인스턴스 생성
- key-pair 생성 : cicd-project-key
- 퍼블릭 IPv4 주소, 키페어로 받음 .pem 파일 비밀번호로 해서 ssh 접속
- java 설치
  - $yum list java* 
  - $sudo yum install java-17-amazon-corretto-devel.x86_64
  
- 이미지 생성 -> 이미지의 AMI 에서 생성된 이미지를 확인할 수 있다.
- EC2 서버 하나더 생성 
  - AMI -> 금방 만들어논 이미지 (java 설치 포함
  - 키페어는 기존에 했던거로.
  - 보안그룹도 기존에 있는거로 사용
  - 인스턴스 4개 생성(jenkins, docker, tomcat, sonarqube)

- 각 서버들끼리 통신할 수 ssh 가능하게 하려면 인바운드 규칙 편집 -> 모든 ICMP-IPv4 , 보안그룹 추가
- $ ping 172.31.34.43 각서버들끼리 통신이 된다.

<br>

### 2. AWS EC2에 Jenkins 서버 설치
___
- EPEL(Extra Packages for Enterprise Linux) : 추가 패키지를 제공하는 리포지토리  
EPEL을 활성화하면 기본 저장소에 없는 소프트웨어(예: 다양한 개발 도구, 라이브러리 등)를 설치할 수 있다.
- Amazon Linux 2023 로 설치를 해서 아래와 같은 명령어로 추가
```text
sudo dnf install -y 'dnf-command(config-manager)'
sudo dnf config-manager --add-repo https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf install epel-release -y

- amazon linux 2 버전일때
$ sudo amazon-linux-extras install epel -y
```

#### Maven , Git 설치
- /opt 폴더로 이동
- $ sudo wget https://dlcdn.apache.org/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz
- $ sudo tar -xvzf apache-maven-3.9.9-bin.tar.gz
- $ sudo mv apache-maven-3.9.9 maven
- bash profile 변경
```text
M2_HOME=/opt/maven
M2=/opt/maven/bin
PATH=$PATH:$M2:$M2_HOME
```

- $ sudo yum install -y git


#### Jenkins 설치
- $ sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
- $ sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
- $ sudo yum install jenkins
- $ sudo systemctl status jenkins
- $ sudo systemctl start jenkins
- 인바운드 규칙 추가 : 사용자 지정 TCP, 8080, anywhere-IPv4, 0.0.0.0/0
- http://13.209.72.58:8080/ 로 jenkins 접속 가능
- $ sudo cat /var/lib/jenkins/secrets/initialAdminPassword 초기비밀번호

<br>

### 3. AWS EC2에 Docker 서버 설치 
___
- $ sudo wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm  
- $ sudo yum install -y docker
- $ sudo systemctl enable docker
- $ sudo systemctl start docker
- $ sudo usermod -aG docker ec2-user : sudo 계속 사용해야해서 ec2-user에 권한 추가

<br>

### 4. AWS EC2에 tomcat 서버 설치
___
- $ sudo wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
- $ sudo wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.98/bin/apache-tomcat-9.0.98.tar.gz
- $ sudo tar -xvzf apache-tomcat-9.0.98.tar.gz
- $ sudo chmod +x ./bin/startup.sh 실행 권한
- $ sudo chmod +x ./bin/shutdown.sh
- http://13.125.227.70:8080/ 톰캣 접속 완료
- $ sudo vi ./webapps/manager/META-INF/context.xml : <Valve 로 시작하는 태그 주석
- $ sudo vi ./webapps/host-manager/META-INF/context.xml
    - 접속 경로를 127.0.0.1 -> 모든 ip로 변경된것. 
- $ sudo vi ./conf/tomcat-users.xml
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

### 5. AWS EC2에 Ansible 서버 설치
___
- $ sudo wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
- $ sudo yum install –y ansible
- $ ssh-keygen
- $ ssh-copy-id ec2-user@[ec2_ip_address]  Tomcat Server, Docker Server
- /etc/ansible/hosts 
```text
[localhost]
localhost

[docker]
172.31.34.43

[tomcat]
172.31.47.76
```
- 키 생성
  - $ssh key-gen -t rsa
  - /home/ec2-user/.ssh/id_rsa.pub 의 내용을 docker, tomcat, localhost(자신) server의 /home/ec2-user/.ssh/authorized_keys 에 복사
  - password 없이 접속 가능하게 된다.
  - ansible docker -m ping 으로 접속 테스트 

<br>

### 6. Jenkins를 이용하여 Tomcat 서버에 배포하기
___
- maven Integration plugin , deploy to container 플러그인 다운
- tool -> add Maven : name - Maven3.9.9 , MAVEN_HOME - /opt/maven
- Item 생성 : My-Tomcat-Project (maven 프로젝트)
  - 소스코드 관리 -> Git 
  - pom.xml , clean compile package –DskipTests=true
  - 빌드후 조치 -> Deploy war/ear to a container : 톰캣 user/pass deployer/deployer 
  - AWS 인스턴스 용량 확장 -> 볼륨을 늘리면 됨.

<br>

### 7. Jenkins를 이용하여 Docker 서버에 배포하기
___
- publish Over ssh 플러그인 설치
- system 에 docker 서버 ssh server 에 추가
  - ip, ec2-user, directory : .
  - id_rsa(private key값) : credential 에 복사
- 젠킨스에서 접속하기 위해 ssh docker서버의 authorized_keys 에 복사


- item 생성
  - Deploy war/ear to a container 사용 x
  - Send build artifacts over SSH : docker-server, source files - target/*.war, remove prefix - target, directory - .
  - docker file 생성
    ```dockerfile
    FROM tomcat:9.0
    COPY ./hello-world.war /usr/local/tomcat/webapps
    ```
  - exec command
    ```text
    docker build --tag=cicd-project -f Dockerfile .;
    docker run -d -p 8080:8080 --name mytomcat cicd-project:latest
    ```

<br>

### 8. Jenkins를 이용하여 Ansible 서버에 배포하기
___
- system 에 ansible 서버 등록 
  - ip는 프라이빗 ip, jenkins 의 ssh 의 id_rsa 를 key에다가 복사
  - jenkins 서버에 id_rsa.pub -> ansible 서버에 authorized_keys 에 복사


- My-Ansible-Project 아이템 생성
  - 빌드 후 조치  
  - 이미지는 도커서버에서 생성, 컨테이너는 ansible 로 생성 예정 
  - add Server (ansible-server - playbook file 가지고 컨테이너 생성만 시킬 예정)
    ```text
    - exec command
    docker build --tag=cicd-project-final -f Dockerfile .;
    exec command ansible-playbook -i hosts create-cicd-devops-container.yml;
    
    
    
    -playbook
    - hosts: all
      #   become: true

      tasks:
        - name: stop current running container
          command: docker stop my_cicd_project
          ignore_errors: yes

        - name: remove stopped cotainer
          command: docker rm my_cicd_project
          ignore_errors: yes

        - name: create a container using cicd-project-ansible image
          command: docker run -d --name my_cicd_project -p 8080:8080 cicd-project-final
    
    - ansible hosts 파일
    /home/ec2-user 경로에 hosts 추가
    [docker]
    172.31.34.43
    ```
