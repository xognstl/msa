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
