## Docker Network and Storage

### 1. Docker Network 
___
- Docker Container는 격리 된 환경에서 작동 -> 다른 Container와 통신 불가능
- Docker Network 사용시 컨테이너 간 통신이 가능


- Docker Network 종류
  - Bridge Network : 하나의 Host 컴퓨터 내에서 여러 컨테이너들이 서로 통신 가능
  - Host Network : Host 컴퓨터와 동일한 네트워크에서 컨테이너를 기동(host 192.168.0.7 , container 192.168.0.6) : 포트 포워딩 없이 내부 애플리케이션 사용
  - None Network : 네트워크를 사용하지 않음, lo 네트워크만 사용, 외부와 단절
  - Overlay Network : 여러 호스트에 분산되어 컨테이너 실행 가능


- Docker Network 명령어
  - docker network ls : Docker Network 목록 보기
  - docker network inspect <NETWORK_NAME> : Docker Network 상세 보기
  - docker network create <NETWORK_NAME> : Docker Network 생성
  - docker network rm <NETWORK_NAME> : Docker Network 삭제
  - docker network connect <NETWORK_NAME> <CONTAINER> : 실행 중인 Container를 Docker Network에 추가
  - docker network disconnect <NETWORK_NAME> <CONTAINER> : Docker Network에 추가 된 Container를 삭제
  - $ docker network connect my-network my-container == $ docker run -it --name my-container --network <NETWORK_NAME> ubuntu:16.04


- $ docker network create --driver bridge my-network : network 생성
- $ docker run -d -p 5000:5000 --restart always --network my-network --name registry registry:2 : 생성한네트워크에 포함
- $ docker run -d -p 8000:8000 --name my-node xognstl/nodejs-demo1:0.2 : 기본 브릿지 네트워크에 컨테이너 추가
- $ docker network connect my-network my-node : 만든 네트워크에도 추가

<br>

### 2. Multi Container 구성
___
- web, db 두가지의 컨테이너를 가지고 서로 같은 네트워크에 연결 
  - $ docker network create --driver bridge my-network
- 데이터 베이스 기동 (name 필수)
  - $ docker run -d -p 13306:3306 --network my-network -e MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=true -e MARIADB_DATABASE=mydb   
    --name my-mariadb edowon0623/my-mariadb:1.0
- web 에서 데이터 베이스 접근시 ip 대신 name 으로 접근 시도 my-mariadb:3306
- 13306:3306 으로 db 생성시 컨테이너간 통신이라 3306 포트 사용
  - $ docker run -d -p 8088:8088 --network my-network -e "spring.datasource.url=jdbc:mariadb://my-mariadb:3306/mydb"   
    --name catalog-service edowon0623/catalog-service:mariadb-demo

<br>

### 3. Docker Volume Storage
___
- Docker Volume : 데이터가 저장되는 Storage, 저장소
- Docker는 개별적인 가상화 환경 : 모든 데이터는 컨테이너 내부에 존재, 컨테이너가 삭제되면 데이터도 삭제
- 컨테이너 내부의 데이터를 외부로 연결 시켜 주는 기능
  - Docker Volume은 기본적으로 /var/lib/docker/volumes/ (Docker Desktop) 디렉토리에 저장
  - host pc, 네트워크에 연결된 드라이브, aws s3 등
  - $ docker volumne ls
  - $ docker volume create my-volume
  - $ docker volume Inspect my-volume
- 컨테이너 생성 시 Volume 연결
  - $ docker run -v <볼륨이름> : <컨테이너 내부 디렉토리 경로> --name <컨테이너 이름> <이미지 이름>

<br>

### 4. Docker Volume Mount 사용
___
- $ docker volume rm volumename1 volumename2 ... 으로 한번에 지울수 있다. 컨테이너에서 사용중인건 안지워짐
- $ docker run -it --network my-network ubuntu:16.04 bash : 우분투 접속
- $ docker inspect 우분투ID : mount 지점이 없다.
- $ docker volume create my-volume 
- $ docker run --rm -it --network my-network -v my-volume:/app/test ubuntu:16.04 bash : 볼륨 추가
- /app/test 경로에 txt 파일 생성 후 bash 종료 (rm 옵션으로 컨테이너가 사라진다.)
- $ docker run --rm -it --network my-network ubuntu:16.04 bash : 아까 만든 app/test/hello.txt 파일이 없다.
- $ docker run --rm -it --network my-network -v my-volume:/app/test ubuntu:16.04 bash : 아까만든 txt 파일이 그위치에 그대로 있다.
- 컨테이너를 삭제해도 my-volume 이라는 볼륨은 그대로 남아있다. 볼륨을 지우지 않는 이상 파일 안없어진다.


- host pc 의 경로를 마운트지점으로 설정
  - $ docker run --rm -it --network my-network -v .\docker_volume_test:/app/test ubuntu:16.04 bash
  - /app/test 경로에 text 파일 생성 시 내 PC의 .\docker_volume_test 지점에 파일이 생긴다.
  - 컨테이너 종료 후 삭제해도 host pc 에 파일은 그대로 남아있다. 다시 컨테이너 실행 할때 볼륨 마운트 옵션을 부여하면 파일이 그대로 있다.


- docker run -d -p 13306:3306 --network my-network -v mariavolume:/var/lib/mysql -e MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=true -e MARIADB_DATABASE=mydb --name my-mariadb edowon0623/my-mariadb:1.0
- maria db 에 볼륨 마운트를 지정해주면 데이터를 보존할 수 있다.
