### 1. 소프트웨어 아키텍처
___
antifragile (깨지기쉬운의 반대) 
- auto scaling : 자동 확장성 , 시스템을 구성하고 있는 인스턴스를 하나의 그룹, 그룹에서 유지되어야하는 최소의 인스턴스를 지정할 수가 있고, 사용량에 따라 자동으로 인스턴스를 증가할 수 있는 환경, 사용량에 따라 자동으로 변경
- microservices : 전체 서비스를 구축하고 있는 개별적인 모듈이나 기능을 독립적으로 개발하고 배포 운영할 수 있도록 세분화된 서비스 
- chaos engineering : 시스템이 급격하고 예측하지 못한 상황이라도 견딜 수 있고 신뢰성을 쌓기 위해 운영 중인 소프트웨어 시스템에 실험하는 방법, 규칙 , 예견되지 않은 불확실성에 대해서도 안정적인 서비스를 제공할 수 있도록 구축 되야 한다.
- Continuous deployments : CI/CD(지속적인 통합, 지속적인 배포) 같은 배포 파이프 라인, 
수십 수백개의 마이크로 서비스를 빌드하고 배포함에 있어서 자동화된 시스템을 구축하고 하나의 작업에서 다른 작업으로 연계되는 과정을 파이프라인으로 연결시킨다.

<br>

### 2. Cloud Native Architecture
___
Cloud Native Architecture 이란 Cloud 환경에서 IT 시스템을 구축하는 것
sacle-up 하드웨어 사양 높히는 것, sacle-out 같은 사양의 서버, 인스턴스를 여러개 배치
클라우드 네이티브에서는 가상서버 기술이 핵심, 서버가상화 , 컨테이너 방식의 가상화 같이 사용
- 확장 가능한 아키텍처
시스템 수평적 확장에 유연, 확장된 서버로 시스템의 부하 분산
- 탄력적 아키텍처
빌드, 배포 시간 단축, 분할된 서비스 구조
- 장애 격리
특정 서비스에 오류가 발생해도 다른 서비스에 영향을 주지 않음

<br>

### 3. Cloud Native Application
___
Cloud Native Application 란 클라우드 네이티브 아키텍처에 의해 설계되고 구현되는 애플리케이션
- 마이크로 서비스로 개발
- CI/CD로 빌드, 테스트, 배포
- 마이크로 서비스에 문제가 발생하였을 경우 바로바로 수정해서 다시 배포하는 과정을 반복하는 형태(Dev Ops)
- 마이크로 서비스들을 클라우드 환경에 배포하고 사용하기 위해 컨테이너 가상화 기술 사용


### 4. 12 Factors
___
Cloud Navtive Application 을 구축함에 있어 고려해야 할 12가지 항목 https://12factor.net

<br>

### 5. Monolithic vs. Microservice
___
- Monolith  : 애플리케이션 개발함에 있어서 필요한 모든 요소를 하나의 커다란 소프트웨어 안에 전부 포함시켜 개발하는방법
- Microservice : 어플리케이션을 구성하는 각각의 구성 요소 및 서비스의 내용을 분리해서 개발하고 운영하는 방식
http 통신을 이용해서 리소스 API 에 통신할 수 있는 작은 규모의 여러 서비스들의 묶음이 모여 하나의 어플리케이션 구성
비즈니스 기능을 중심으로 구축되어야 하고 자동화된 배포 시스템을 사용해야한다.
구분된 서비스들은 독립적인 언어와 DB를 사용 할 수 있다.

<br>

### 6. Microservice Architecture란?
___
- 마이크로 서비스들이 가지고 있는 환경에 대한 정보, 설정 정보는 코드내가 아닌 외부에 있는 시스템을 통해서 관리
- RESTful 권장
- 서비스를 제공하는 인스턴스들은 부하 분산 처리나 스케일 업, 다운등 을 동적으로 처리할 수 있도록 구성
- 시각화 할수 있는 관리도구가 있어야 한다.
- CI/CD

<br>

### 7. SOA (Service-Oriented Architecture) vs MSA
___
- SOA : 재사용을 통한 비용 절감
- MSA : 서비스 간의 결합도를 낮추어 변화에 능동적으로 대응 , 각독립된 서비스가 노출된 rest api를 사용

<br>

### 8. Microservice Architecture Structures
___
마이크로서비스는 독립적으로 배포되고 확장될 수 있는 서비스를 조합해서 하나의 어플리케이션을 구성하는 아키텍처 패턴

<br>

### 9. Spring Cloud란?
___
– 환경 설정 관리
Spring cloud Config Server
– location transparency(위치 투명성) 
사용자는 시스템 내 리소스가 물리적 으로 어디에 위치 하는지 알 수 없도록 하는 기능 , 서비스의 등록, 위치정보 확인, 검색 등 서비스를 위해 넷플릭스 유레카 서버 사용 
Naming Server(Eureka)
– Load Balncing, gateway
Ribbon (Client Side)
Spring Cloud Gateway
–  Easier REST Clients
FeignClient
– Visibility and monitoring
Zipkin Distributed Tracing (분산 추적)
Netflix API gateway 
– Fault Tolerance (회복성 패턴)
Hystrix

<br>

### 설치 SW
IntelliJ , Git, Visual Studio Code, Postman
