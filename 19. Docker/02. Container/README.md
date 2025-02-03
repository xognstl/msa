## Docker Essentials - Container

### 1. Docker Image
___
- 이미지 : docker 를 통해 실행하고자 하는 OS, 미들웨어, 서비스 어플리케이션 같은 서비스 형태가 제공대기 위한 하나의 템플릿
  - Layer 저장 방식 : 여러개의 layer 를 하나의 파일 시스템으로 사용
- Dockerfile : docker image 를 생성하기 위한 스크립트 파일
  - 자체 DSL 언어 사용 -> 이미지 생성과정 기술
  - 서버에 프로그램을 설치하는 과정을 dockerfile 로 기록 관리
  - 소스와 함께 버전관리가 되며, 누구나 수정 가능

<br>

### 2. Docker Container
___
- Container(인스턴스) : Docker Image 를 실행한 상태
  - 격리 된 시스템자원 및 네트워크를 사용할수 있는 독립적인 실행 단위
  - 읽기 전용 상태인이미지에 변경 된 사항을저장할수 있는 컨테이너 계층에저장
#### 명령어
- $docker run [OPTIONS]IMAGE[:TAG|@DIGEST] [COMMAND] [ARG…]
  - run : create , start, running 를 한번에 할 수 있다. 
  - -d, --detach : Detached mode, Background mode
  - -p, --publish : 호스트와 컨테이너의 포트를 연결(포워딩)
  - -v, --volume : 호스트와 컨테이너의 디렉토리를 연결(마운트)
  - -e, --env : 컨테이너내에서 사용할 환경변수 설정
  - --name : 컨테이너 이름 설정
  - --rm : 프로세스 종료시 컨테이너 자동 제거
  - -it : -i와 –t를 동시에 사용한것으로 터미널 입력을 위한 옵션
  - -link : 컨테이너연결[컨테이너명:별칭]
