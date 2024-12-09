## API Gateway Service

### 1. API Gateway 란?
___
#### API Gateway Service : 
사용자나 외부 시스템으로부터 요청을 단일화하여 처리할 수 있도록 해준 것.  
사용자가 설정한 라우팅 설정에 따라서 각각 엔드포인트로 클라이언트를 대신해서 요청하고 응답을 받으면  
다시 클라이언트한테 전달해주는 Proxy 역할을 하게 된다.  
시스템의 내부 구조는 숨기고 외부의 요청에 대해서 적절한 형태로 가공해서 응답할 수 있다는 장점이 있다.

- API Gateway Service 특징
  - 인증 및 권한 부여
  - 서비스 검색 통합
  - 응답 캐싱 저장
  - 정책, 회로 차단기 및 QoS 다시 시도
  - 속도 제한
  - 부하 분산 (Load Balancing)
  - 로깅, 추적, 상관관계
  - 헤더, 쿼리 문자열 및 청구 변환
  - IP 허용 목록에 추가


#### Netflix Ribbon
- Spring cloud 에서의 msa 간 통신
1) RestTemplate : 객체생성해서 ip, port를 이용
2) Reign Client : 인터페이스를 이용해 마이크로 서비스 이름을 등록해서 이용

- Ribbon : client side load balancer, 비동기 X, 
  - 마이크로 서비스의 이름 가지고 클라이언트 안에서 필요한 데이터 호출 가능
  - 헬스 체크 : 서비스 정상 작동 여부 확인

#### Netflix Zuul
- Routing, Gateway 역할
- client <-> Netflix Zuul <-> first, second Service

<br>

### 2. Netflix Zuul - 프로젝트 생성 (Deprecated)
___

#### first service, second service
- spring boot 2.3.8 
- dependency : lombok, spring web, eureka discovery client  
** Zuul 이라는 게이트웨이에서 사용자 요청이 들어왔을때 두가지의 서비스로 잘 분산되서 전달되는지 확인 할 것이다!

#### Zuul Service
- spring boot 2.3.8
- dependency : lombok, Spring Web, Zuul


- API Gateway 의 장점 비즈니스 서비스 로직에 사전 처리 , 사후처리 사용가능(인증, 로깅) 필터로 구현가능

- First, Second Service 프로젝트 생성
```yaml
server:
  port: 8081
spring:
  application:
    name: my-first-service

eureka:
  client:
    fetch-registry: false
    register-with-eureka: false
# second 서비스는 port 8082 , name : my-second-service
```

```java
@RestController
@RequestMapping("/")
public class FirstServiceController {
    @GetMapping("/welcome")
    public String welcome() {
        return "welcome to the First service";
    }
}
// second 서비스는 return Second , class 명 SecondServiceController 
```
http://127.0.0.1:8081/welcome 접속 시 welcome to the First service 출력


- Zuul Service 프로젝트 생성
```java
@SpringBootApplication
@EnableZuulProxy
public class ZuulServiceApplication {
}
```

```yaml
server:
  port: 8000
spring:
  application:
    name: my-zuul-service
zuul:
  routes:
    first-service:
      path: /first-service/**
      url: http://localhost:8081
    second-service:
      path: /second-service/**
      url: http://localhost:8082
```
http://127.0.0.1:8000/first-service/welcome => 접속 시 welcome to the First service 출력  
http://127.0.0.1:8000/second-service/welcome => 접속 시 welcome to the Second service 출력

* 8000번이라는 API Gateway 역할을 해주는 zuul을 이용해 first, second service 사용자의 요청에 따라 어떤 마이크
로서비스를 이용할지 정할 수 있다. zuul 이 해주는 가장 기본적인 gateway 라우팅

<br>
