## E-commerce 애플리케이션


### 1. E-commerce 애플리케이션 개요
___

- CATALOG-SERVICE(사용자 -> 상품 조회, ORDER-SERVICE -> 상품 수량 업데이트)
- USER-SERVICE(사용자 -> 사용자 조회, 주문 확인)
- ORDER-SERVICE(사용자 -> 상품 주문, USER-SERVICE -> 주문 조회)
- 상품 수량 업데이트는 메세지 서비스(Kafka 이용)

<br>

### 2. E-commerce 애플리케이션 구성
___
- Registry Service(Eureka Server)
- Catalog-Service, User-Service, Order-Service 는 Eureka Service 에 등록
- Catalog-Service 와 Order-Service 하고 서로 데이터를 주고 받기 위한 용도로 메시지 큐잉서버(Kafka)
- 외부에서 클라이언트 요청이 들어왔을 때 Spring Cloud API Gateway 라우팅 서비스
- Configuration Service(Config Server) : 3가지 마이크로서비스가 가지고 있어야 될 환경설정 정보를 여기에 등록해 놓고 참조해서 사용

#### 애플리케이션 구성 요소
- Git Repository : 마이크로서비스 소스 관리 및 프로파일 관리
- Config Server : Git 저장소에 등록된 프로파일 정보 및 설정 정보
- Eureka Server : 마이크로서비스 등록 및 검색
- API Gateway Server : 마이크로서비스 부하 분산 및 서비스 라우팅
- Microservices : 회원 서비스 , 주문 서비스 , 상품 서비스
- Queuing System : 마이크로서비스 간 메시지 발행 및 구독

#### 애플리케이션 API
- Catalog-Service 
  - GET /catalog-service/catalogs : 상품 목록 제공
- User-Service
  - POST /user-service/users : 사용자 정보 등록
  - GET /user-service/users : 사용자 정보 조회
  - GET /user-service/users/{user_id} : 사용자 정보, 주문 내역 조회
- Order-Service
  - POST /order-service/users/{user_id}/orders : 주문 등록
  - GET /order-service/users/{user_id}/orders : 주문 확인
