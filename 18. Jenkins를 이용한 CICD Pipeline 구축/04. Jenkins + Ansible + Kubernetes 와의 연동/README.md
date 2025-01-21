## Jenkins + Ansible + Kubernetes 와의 연동

### 1. Kubernetes
___
- Kubernetes(k8s)
  - 오픈소스 기반의 컨테이너화 된 애플리케이션(워크로드와 서비스)의 자동배포, 스케일링 등을 제공하는 관리 플랫폼
  - 컨테이너 가상화 기술을 사용함에 있어 각각의 컨테이너를 관리, 스케줄링 도구 

- 장점
  - 컨테이너화 된 애플리케이션 구동
  - 서비스 디스커버리와 로드 밸런싱, 스토리지 오케스트레이션
  - 자동화된 롤아웃과 롤백, 빈 패킹(bin packing), 복구(self-healing)
  - 시크릿과 구성 관리
- 단점
  - 소스 코드 배포 X, 빌드 X
  - 애플리케이션 레벨 서비스 X
  - 로깅, 모니터링 솔루션 X
  - 포괄적인 머신 설정, 유지보수, 관리, 자동 복구 시스템을 제공 X

- Kubernetes Cluster
  - Master Node : 전체 구성되어있는 각각의 노드를 관리, 설정정보, 스케줄, api 처리할 수 있도록 구성
  - Node : 컨테이너 자체를 운영하고 스케줄링해주는 작업을 한다.   
  container를 관리 하기 위한 Pod, Pod를 운영하기 위한 Kubelet 이 있다.
  - 클라이언트가 명령을 전달하면 master node 의 api서버에 전달되고 각각 노드에 전달.   
  전달 받는 역할을 각 노드의 k8s Proxy 가 해준다. 네트워크에 대한 유지, 관리 
  - 사용자들은 각각의 노드를 통해 서비스를 사용할 수 있다.
  - 각각의 노드는 컨테이너 엔진이 있어야 하고, Pod는 여러가지 컨테이너가 묶여서 하나의 Pod가 된다.

- CI/CD 파이프라인을 통해 kubernetes에 배포한다는 것은 컨테이너 형태로 배포를 하는데 그 컨테이너는 Pod형태로 감싸서 배포
- 감싸져있는 결과물을 외부에서 사용할 수 있도록 서비스라는 오브젝트가 붙어서 사용할 수 있는 상태를 만들어줄 것이다.

<br>

### 2. Kubernetes 기본 명령어
___
- master 는 사용자요청하는 명령어 처리 , work node 에선 컨테이너 작업 진행

#### Minikube 설치
- docker desktop 설정 -> kubernetes -> enable kubernetes 체크
- $kubectl get nodes:  현재 노드 조회 , minikube 라 단일 노드만 있다.(master, node 둘다 됨)
  NAME             STATUS   ROLES           AGE     VERSION
  docker-desktop   Ready    control-plane   8m55s   v1.30.5
- $kubectl get pods : pod 조회
- $kubectl get deployments
- $kubectl get services

#### Nginx 설치 
- $kubectl delete pod/sample-nginx : 삭제
- $kubectl delete deployment.apps/sample-nginx : deployment 삭제

1) 방법
   - $kubectl run sample-nginx --image=nginx --port=80 : 설치 하면 pods 가 생성 된다.
   - $kubectl describe pods sample-nginx 
2) deployment 이용해서 설치 
   - Pod 를 레플리카 세트라고 해서 여러 개의 형태로 스케일링해서 만들거나 스케줄링 작업, 히스토리 작업할때 사용할 수 있는 설치 개념
   - Pod 의 상위 개념으로 배포 단위를 deployment 로 감싸서 배포
   - $kubectl create deployment sample-nginx --image=nginx
   - pod 를 삭제해도 다른 pod 가 생성 된다. => deploy 형태로 실행하면 최소한의 pod 개수를 유지하려한다.
   - $kubectl scale deployment sample-nginx --replicas=2 => 유지하려는 pod의 개수 2개로 늘림
   - deployment 조회 하면 READY 2/2 가 된다.
3) 스크립트 생성하여 설치 (가장 많이 사용)
   - $kubectl apply -f sample1.yml 
 ```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

<br>

### 3. Kubernetes Script 파일
___
- $ kubectl get pods -o wide : Pods 의 좀더 자세한 정보
- 컨테이너 자체에 추가 명령이나, bash shell 접속 
- $kubectl exec -it nginx-deployment-77d8468669-5sdcj -- /bin/bash
- $apt-get update, $apt-get install -y curl wget, $hostname -i
- $curl -X GET http://10.1.0.11 하면 nginx 페이지 보인다. 외부에서 사용할 수 있게 오픈해야함. 
- $kubectl expose deployment nginx-deployment --port=80 --type=NodePort
- nginx-deployment   NodePort    10.111.72.30   <none>        80:32351/TCP   4s : 서비스 검색시 
- window 에서 http://localhost:32351/ 하면 페이지 확인할 수 있다.

#### 서비스 올리기
- cicd-devops-deployment.yml , image 에 hub 사이트에 올린 image 를 다운받아 kubernetes 상태로 기동
- $kubectl apply -f cicd-devops-deployment.yml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cicd-deployment
spec:
  selector:
    matchLabels:
      app: cicd-devops-project
  replicas: 2

  template:
    metadata:
      labels:
        app: cicd-devops-project
    spec:
      containers:
      - name: cicd-devops-project
        image: xognstl/cicd-project-ansible
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```
- cicd-devops-service.yml, 외부에서 접근할 수 있게 서비스 등록
- $kubectl apply -f cicd-devops-service.yml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: cicd-service
  labels:
    app: cicd-devops-project
spec:
  selector:
    app: cicd-devops-project
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32000
```
- http://localhost:32000/hello-world/ 하면 쿠버네티스에서 서비스되는것을 확인할 수 있다.

<br>

### 4. Kubernetes + Ansible 연동
___
- Ansible Server -> K8s Master 로 접속이 되면 ansible 에서 playbook 파일 실행   
-> playbook 에서는 k8s 에 있는 yml 파일을 실행 하는 작업을 할것이다.


- Ansible Server) k8s-master 접속 테스트
  - $ssh dell@172.10.72.88 : ansible 서버에서 내 host pc 로 접속 
- Ansible Server) k8s/hosts 파일 작성
  - k8s 디렉토리 생성후 hosts 파일 생성
```text
[ansible-server]
localhost

[kubernetes]
172.10.72.88
```
- Ansible Server) ssh-keygen, ssh-copy-id
  - $ssh-copy-id dell@172.10.72.88
  - $ ansible -i ./k8s/hosts kubernetes -m ping
  - $ ansible -i ./k8s/hosts kubernetes -m win_ping
  - window 에서 ping 안되는거 아래와 같은 방법으로 해결
  - https://www.inflearn.com/community/questions/686434/ssh-copy-id-%EC%97%90%EB%9F%AC-%EB%AC%B8%EC%9D%98

- K8s Master) create a deployment yaml file
  - cicd-devops-deployment.yml
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: cicd-deployment
    spec:
      selector:
        matchLabels:
          app: cicd-devops-project
      replicas: 2
  
      template:
        metadata:
          labels:
            app: cicd-devops-project
        spec:
          containers:
          - name: cicd-devops-project
            image: xognstl/cicd-project-ansible
            imagePullPolicy: Always
            ports:
            - containerPort: 8080
    ```
- Ansible Server) create a playbook file for deployment
  - k8s-cicd-deployment-playbook.yml, command => win_command 로 변경 
    ```yaml
    - name: Create pods using deployment
      hosts: kubernetes
    - name: create a deployment
      win_command: kubectl apply -f E:\cicd\k8s\cicd-devops-deployment.yml
    ```
- Ansible Server) execute a playbook file (for deployment)
  - $ansible-playbook -i ./k8s/hosts k8s-cicd-deployment-playbook.yml
- Ansible Server) execute a playbook file (for service)
  - k8s-cicd-service-playbook.yml 
```yaml
- name: create service for deployment
  hosts: kubernetes
  tasks:
  - name: create a service
    win_command: kubectl apply -f E:\cicd\k8s\cicd-devops-service.yml
```
  - $ansible-playbook -i ./k8s/hosts k8s-cicd-service-playbook.yml
  - 실행시 32000 포트의 서비스가 생성됨을 확인 할 수 있다.
  - http://localhost:32000/hello-world/ 접속 

<br>

### 5. Jenkins + Ansible + Kubernetes 연동하기
___
#### Jenkins + k8s
- Jenkins 의 system -> publish over ssh 에 k8s 추가
- My-K8s-Project Item 생성 
  - build Triggers : SSH-Server : k8s-master
  - exec command : $kubectl apply -f E:\cicd\k8s\cicd-devops-deployment.yml
  
#### Jenkins + Ansible + K8s
- My-K8s-Project-using-Ansible Item 생성
  - Build Triggers : SSH Server: ansible-server
  - Exec command : 
    - $ansible-playbook -i ./k8s/hosts k8s-cicd-deployment-playbook.yml;
    - $ansible-playbook -i ./k8s/hosts k8s-cicd-service-playbook.yml; 
    - 해당 playbook 파일에 ansible -> k8s 의 yml 파일 호출하는것이 포함되어 있다.
   
<br>

### 6. 전체 CI/CD 자동화 프로세스 구성
___
- CI(Continuous Integration, 지속적인 통합)
  - Git pull, 단위테스트, 패키징
  - docker image 생성
  - docker hub registry 에 이미지 push
  - local 이미지 삭제
- CD(Continuous Delivery, Deployment)
  - 등록된 registry 를 받아서 k8s deployment 생성(2개 pod 생성)
  - 서비스 생성

- 최신 코드를 가져와 이미지를 만드는 작업
  - create-cicd-project-image-playbook.yml(playbook 형태)
    - docker build
    - docker hub 에 push
    - local image 삭제
```yaml
- hosts: all
  tasks:
    - name: create a docker image with deployed waf file
      command: docker build -t xognstl/cicd-project-ansible .
      args:
        chdir: /root

    - name: push the image on Docker Hub
      command: docker push xognstl/cicd-project-ansible

    - name: remove the docker image from the ansible server
      command: docker rmi xognstl/cicd-project-ansible
      ignore_errors: yes
```
  - Dockerfile
```dockerfile
FROM tomcat:9.0
COPY ./hello-world.war /usr/local/tomcat/webapps
```

- Jenkins Item 생성 : My-K8s-Project-for-CI 
  - Poll SCM : * * * * *
  - Post-build Actions : SSH-Server : ansible-server
  - Exec command : ansible-playbook -i ./k8s/hosts create-cicd-project-image-playbook.yml --limit ansible-server

- Post-build actions(빌드 후 조치)
  - build other projects -> My-K8s-project-using-Ansible(deployment, service 생성)

- deployment 생성 하는 코드에 deployment 삭제 코드 추가
```yaml
- name: Create pods using deployment 
  hosts: kubernetes 
  # become: true
  # user: ubuntu
 
  tasks: 
  - name: delete the previous deployment
    win_command: kubectl delete E:\cicd\k8s\deployment.apps/cicd-deployment
    ignore_errors: yes

  - name: create a deployment
    win_command: kubectl apply -f E:\cicd\k8s\cicd-devops-deployment.yml
```
