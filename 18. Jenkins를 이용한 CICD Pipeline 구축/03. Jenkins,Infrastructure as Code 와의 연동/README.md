## Jenkins + Infrastructure as Code 와의 연동

### 1. Infrastructure as Code 개요와 Ansible의 이해
___
#### Infrastructure as Code
- 시스템, 하드웨어 또는 인터페이스의 구성정보를 파일(스크립트)을 통해 관리 및 프로비저닝    
=> 코드에 의해서 Infrastructure 를 관리하고 생성, 네트워크 설정 관리
- 버전 관리를 통한 리소스 관리
- Terraform, Ansible, AWS CloudFormation, CHEF, puppet 등

#### Ansible
- 도커에서 기동되었던 컨테이너를 다시 기동하거나 이미지를 다시 배포하거나 하는 용도로 사용(Configuration Management, Deployment)
- 여러 개의 서버를 효율적으로 관리할 수 있게 해주는 환경 구성 자동화 도구
  - Configuration Management, Deployment & Orchestration tool
  - IT infrastructure 자동화
- Push 기반 서비스
- Simple, Agentless(관리할 서버, main서버)

- 할수 있는일
  - 설치(apt-get, yum, homevrew), 파일, 스크립트 배포(copy), 다운로드(get_url, git), 실행(shell, task)
  - 결과 ok/failed/changed/unreachable

- 환경 설정 파일 : /etc/ansible/ansible.cfg 
- Ansible에서 접속하는 호스트 목록 : /etc/ansible/hosts

<br>

### 2. Docker 컨테이너로 Ansible 실행하기
___
- $docker pull ansibleImage(hub에서 찾아서)  : 이미지 다운 (docker + Ansible)
- $docker run --privileged --name ansible-server -itd -p 20022:22 -p 8083:8080 -e container=docker -v /sys/fs/cgroup:/sys/fs/cgroup edowon0623/ansible:latest /usr/sbin/init
- $ssh root@localhost -p 20022 : ansible 서버 접속
- [root@52a39c6a2ba9 ~]#systemctl start docker : docker 실행
- [root@52a39c6a2ba9 ~]#ansible --version
- [root@52a39c6a2ba9 ~]#vi /etc/ansible/hosts
```text
[devops]
172.17.0.3  (docker)
172.17.0.4  (ansible)
```
- 터미널에 docker network inspect bridge 에보면 docker, ansible등 서버들 연결 되어있다.


- Ansible 서버에서 등록했던 3,4 서버 접속 테스트 
- ssh 로 접속하면 계속 root/password 쳐야한다. 그래서 Ansible 서버에서 만들어져 있는 키값을 가지고 3,4번서버 접속  
.현재 디렉토리에 있는 id_rsa 라는 public , private 키를 생성 하고 상대방에 배포를 시켜줌으로 접속을 할 수 있다.
- [root@52a39c6a2ba9 ~]#ssh-keygen : 키값 생성 
- [root@79b08c505773 ~]# ssh root@172.17.0.3 : 키값 배포전에 접속 해서 password 입력
- [root@52a39c6a2ba9 ~]#ssh-copy-id root@172.17.0.3 : 배포 하는것
- 배포후에 다시 3번 서버에 접속하면 password 없이 접속 가능, 4번서버에도(자기자신) 똑같이 키값 배포

#### 실행 옵션
- -i (--inventory-file) : 적용 될 호스트들에 대한 파일 정보
- -m (--module-name) : 모듈 선택
- -k (--ask-pass) : 관리자 암호 요청
- -K (--ask-become-pass) : 관리자 권한 상승
- --list-hosts : 적용되는 호스트 목록
- 멱등성 : 같은 설정을 여러 번 적용하더라도 결과가 달라지지 않는 성질

<br>

### 3. Ansible 모듈 사용
___
- https://docs.ansible.com/ansible/2.9/modules/list_of_all_modules.html
- $ansible all -m ping : Ansible Test
```text
172.17.0.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
172.17.0.3 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
```
- $ ansible all -m shell -a “free -h” : 디스크 용량 확인
- $ ansible all -m copy -a “src=./test.txt dest=/tmp” 파일 전송
  - $ touch test.txt : 파일 생성 
- $ ansible all -m yum -a “name=httpd state=present” 서비스 설치

<br>

### 4. Ansible Playbook 사용하기
___
- 사용자가 원하는 내용을 미리 작성해 놓은 파일
  - 설치, 파일 전송, 서비스 재시작, 다수의 서버에 반복 작업을 처리하는 경우
- Playbook
  - $ vi first-playbook.yml 작성
```yaml
---
- name: Add an ansible hosts
  hosts: localhost
  tasks:
    - name: Add an ansible hosts
      blockinfile:
        path: /etc/ansible/hosts
        block: |
          [mygroup]
          172.17.0.5
```
  - $ ansible-playbook first-playbook.yml
  - $ cat /etc/ansible/hosts
```text
[devops]
172.17.0.3
172.17.0.4
# BEGIN ANSIBLE MANAGED BLOCK
[mygroup]
172.17.0.5

```

#### 파일 복사 방법
```yaml
---
- name: Ansible Copy Example Local to Remote
  hosts: devops
  tasks:
    - name: copying file with playbook
      copy:
        src: ~/sample.txt
        dest: /tmp
        owner: root
        mode: 0644
```
- $ansible-playbook playbook-sample1.yml
- 3, 4번 서버에 /tmp에 sample.txt 복사가 됬다.

####
```yaml
---
- name: Download Tomcat9 from tomcat.apache.org
  hosts: devops
  #become: yes
  # become_user: root
  tasks:
    - name: Create a Directory /opt/tomcat9
      file:
        path: /opt/tomcat9
        state: directory
        mode: 0755
    - name: Download the Tomcat checksum
      get_url:
        url: https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.98/bin/apache-tomcat-9.0.98.tar.gz.sha512

        dest: /opt/tomcat9/apache-tomcat-9.0.98.tar.gz.sha512
    - name: Register the checksum value
      shell: cat /opt/tomcat9/apache-tomcat-9.0.98.tar.gz.sha512 | grep apache-tomcat-9.0.98.tar.gz | awk '{ print $1 }'
      register: tomcat_checksum_value
    - name: Download Tomcat using get_url
      get_url:
        url: https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.98/bin/apache-tomcat-9.0.98.tar.gz
        dest: /opt/tomcat9
        mode: 0755
        checksum: sha512:{{ tomcat_checksum_value.stdout }}"

```
- tomcat 디렉토리 생성, 톰켓 다운로드
