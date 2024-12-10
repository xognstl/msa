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

### 3. Netflix Zuul - Filter 적용 (Deprecated)
___

```java
package com.example.zuulservice.filter;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;


@Slf4j
@Component
public class ZuulLoggingFilter extends ZuulFilter {

//    Logger logger = LoggerFactory.getLogger(ZuulLoggingFilter.class);
    // lombok @Slf4j
    @Override
    public Object run() throws ZuulException {  //실제동작
        log.info("************************ printing logs : ");
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest(); //servlet 에서 사용자 요청 정보 처리
        log.info("************************ " + request.getRequestURI());

        return null;
    }
    @Override
    public String filterType() {
        return "pre";   // 사전 필터 (사후는 post)
    }
    @Override
    public int filterOrder() {
        return 1;   // 순서
    }
    @Override
    public boolean shouldFilter() {
        return true;    //사용 여부
    }
}
```
http://127.0.0.1:8000/second-service/welcome => zuul-service에서 로그 확인 가능

<br>

### 4. Spring Cloud Gateway 란?
___

- Spring cloud Gateway(gateway-service) 
- dependencies : DevTools, Eureka Discovery Client, Gateway
- 비동기 처리가 가능하다 (zuul 은 안됨.)

<br>

### 5. Spring Cloud Gateway - 프로젝트 생성
___

```yaml
server:
  port: 8080
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://localhost:8761/eureka
spring:
  application:
    name: apigateway-service
  cloud:
    gateway:
      routes: 
        - id: first-service
          uri: http://localhost:8081/
          predicates:
            - Path=/first-service/**
        - id: second-service
          uri: http://localhost:8082/
          predicates:
            - Path=/second-service/**
```
- 기동하면 Netty 로 기동된다. 비동기 ( tomcat x)
```java
@RestController
@RequestMapping("/first-service")
public class FirstServiceController {
} // second 도 같이 requestmapping 변경
```
http://localhost:8080/second-service/welcome => 정상적으로 second 서비스 불러온다.

<br>

### 6.Spring Cloud Gateway - Filter 적용
___
- Client <-> spring cloud gateway <-> first, second service
- 클라이언트가 spring cloud gateway에 어떤 요청을 전달하게 되면 first로갈지 second로 갈지 판단한 다음에 서비스 요청을 분기해준다.
- spring cloud gateway 에서 요청이 들어오면(gateway handler mapping) 그게 어떤것인지 판단하고(predicate) 사전 필터, 사후필터(pre, post filter)
- 작업은 property , java code 로 할 수 있다.

<br>
- java code로 라우팅 정보를 등록할 예정이니 cloud 관련 application.yml 파일은 잠시 주석 처리

```java
@Configuration
public class FilterConfig {
    @Bean
    public RouteLocator gatewayRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
                .route(r -> r.path("/first-service/**")
                        .filters(f -> f.addRequestHeader("first-request", "first-request-header")
                                .addResponseHeader("first-response", "first-response-header"))
                        .uri("http://localhost:8081"))
                .route(r -> r.path("/second-service/**")
                        .filters(f -> f.addRequestHeader("second-request", "second-request-header")
                                .addResponseHeader("second-response", "second-response-header"))
                        .uri("http://localhost:8082"))
                .build();
    }
}
```
- r 값이 전달되게 되면 패스를 확인 하고 필터적용해서 uri로 이동 시켜준다. 사용자로부터 First service라는 요청이 들어온다.
- 그러면 url 로 이동, 중간에 request filter addRequestHeader("first-request", "first-request-header")
- response filter 에는  .addRequestHeader("first-response", "first-response-header") 를 추가한다. 
- yml에 있던 설정과 똑같은것을 자바로 한거다.

```java
    @GetMapping("/message")
    public String message(@RequestHeader("first-request") String header){
        log.info(header);
        return "hello world in First Service";
    }   //second service 에도 똑같이 메세지 남겨준다.
```

http://localhost:8080/first-service/message =>  
INFO 37768 --- [nio-8081-exec-2] c.e.firstservice.FirstServiceController  : first-request-header 로그 확인 가능  
response 에도 first-response: first-response-header 개발자 도구에서 확인가능

<br>

** application.yml 파일로 금방과 같은 작업을 한다. FilterConfig.java Configuration, bean 주석  

```yaml
spring:
  application:
    name: apigateway-service
  cloud:
    gateway:
      routes:
        - id: first-service
          uri: http://localhost:8081/
          predicates:
            - Path=/first-service/**
          filters:
            - AddRequestHeader=first-request, first-request-header2
            - AddResponseHeader=first-response, first-response-header2
        - id: second-service
          uri: http://localhost:8082/
          predicates:
            - Path=/second-service/**
          filters:
            - AddRequestHeader=second-request, second-request-header2
            - AddResponseHeader=second-response, second-response-header2
```

- http://localhost:8000/first-service/message => 헤더 확인 가능

<br>

### 7. Spring Cloud Gateway - Custom Filter 적용
___

- 사용자 정의 필터 : 로그 출력, 인증 처리, 로케일 변경 등에 사용
- CustomFilter 는 AbstractGatewayFilterFactory 상속 받아서 사용
- 구현 메소드 apply() 

<br>

- CustomFilter class 생성
```java
@Component
@Slf4j
public class CustomFilter extends AbstractGatewayFilterFactory<CustomFilter.Config> {
    public CustomFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        // Custom Pre Filter
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();

            log.info("Custom PRE Filter  : request id -> {} ", request.getId());

            //Custom Post Filter
            return chain.filter(exchange).then(Mono.fromRunnable(()->{
                log.info("Custom Post filter : response code -> {}", response.getStatusCode());
            }));
        };
    }

    public static class Config {
        // configuration 정보 입력
    }
}
```
- 첫번째 매개변수인 exchange 를 기자고 request, response 객체를 가져오고 처리가 끝나고 post filter 추가 할 수 있다.
- 요청이 들어오면 사전 필터에서 요청 ID를 로그에 출력된다.
- 요청이 체인(chain)을 따라 다른 필터나 백엔드 서비스로 전달된다.
- 백엔드 서비스에서 응답을 받은 후 사후 필터가 실행되어 응답 상태 코드를 로그 출력

<br>

- yaml 파일 수정
```yaml
          filters:
#            - AddRequestHeader=first-request, first-request-header2
#            - AddResponseHeader=first-response, first-response-header2
            - CustomFilter
```
- first, second service controller 추가
- /message 는 헤더가 필요해서 check controller 추가한것.
<br>

```java
    @GetMapping("/check")
    public String check() {
        return "Hi, this is a message from First Service";
    }
```

- http://localhost:8000/first-service/check 호출 시 로그 확인 가능

Custom PRE Filter  : request id -> 14e5a4d2-1  
Custom Post filter : response code -> 200 OK 

<br>
*** pre filter 먼저 동작, chain filter 값을 반환시켜줌으로써 post filter 동작을 구현  

<br>

### 8. Spring Cloud Gateway - Global Filter
___

- 글로벌 필터는 어떠한 라우터 정보가 실행 된다 해도 공통적으로 다 시행 될 수 있다.
- 커스텀 필터는 원하는 라우터 정보에다가 개별적으로 다 등록 해야 한다.  

-GlobalFilter.java
```java
@Override
public GatewayFilter apply(Config config) {
    return (exchange, chain) -> {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();

        log.info("Global Filter baseMessage : {} ", config.getBaseMessage());

        if (config.isPreLogger()) {
            log.info("Global Filter Start : request id -> {} ", request.getId());
        }
        return chain.filter(exchange).then(Mono.fromRunnable(()->{
            if (config.isPostLogger()) {
                log.info("Global Filter End : response code -> {} ", response.getStatusCode());
            }
        }));
    };
}

@Data
public static class Config {
    private String baseMessage;
    private boolean preLogger;
    private boolean postLogger;
}
```
-application.yml
```yaml
  cloud:
    gateway:
      default-filters:
        - name: GlobalFilter
          args:
            baseMessage: Spring cloud Gateway Global Filter
            preLogger: true
            postLogger: true
```
- http://localhost:8000/first-service/check 실행시  호출 시 로그 확인 가능  
Global Filter baseMessage : Spring cloud Gateway Global Filter  
Global Filter Start : request id -> bd2f928e-1  
Custom PRE Filter  : request id -> bd2f928e-1  
Custom Post filter : response code -> 200 OK  
Global Filter End : response code -> 200 OK   

<br>

### 9. Spring Cloud Gateway - Logging Filter
___

- LoggingFilter.java
```java
    @Override
    public GatewayFilter apply(Config config) {
        GatewayFilter filter = new OrderedGatewayFilter((exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();

            log.info("Logging Filter baseMessage : {} ", config.getBaseMessage());

            if (config.isPreLogger()) {
                log.info("Logging Pre Filter : request id -> {} ", request.getId());
            }
            return chain.filter(exchange).then(Mono.fromRunnable(()->{
                if (config.isPostLogger()) {
                    log.info("Logging Post Filter : response code -> {} ", response.getStatusCode());
                }
            }));
        }, Ordered.HIGHEST_PRECEDENCE);

        return filter;
    }

    @Data
    public static class Config {
        private String baseMessage;
        private boolean preLogger;
        private boolean postLogger;
    }
```
- yml 파일에 LoggingFilter, 파라미터 등록
```yaml
          filters:
            - name: CustomFilter # 추가적인 파라미터 넣으려면 name 을 넣어줘야한다.
            - name: LoggingFilter
              args:
                baseMessage: Hi Logging filter
                preLogger: true
                postLogger: true
```
- filter 실행순서 
- Gateway client <-> Gateway Handler <-> Global filter <-> Custom Filter <-> Logging Filter <-> Proxied Service
- http://localhost:8000/second-service/check 실행시 아래와 같은 로그 출력, Logging 이 먼저 실행되는건 Ordered.HIGHEST_PRECEDENCE 떄문
  
Logging Filter baseMessage : Hi Logging filter  
Logging Pre Filter : request id -> 4b6bee7d-3  
Global Filter baseMessage : Spring cloud Gateway Global Filter  
Global Filter Start : request id -> 4b6bee7d-3  
Custom PRE Filter  : request id -> 4b6bee7d-3  
Custom Post filter : response code -> 200 OK  
Global Filter End : response code -> 200 OK  
Logging Post Filter : response code -> 200 OK  

<br>

### 10. Spring Cloud Gateway - Load Balancer
___

#### Spring Cloud Gateway - Eureka 연동  
- Eureka Server: Service Discovery, Registration
- /first-service/welcome 호출시 API Gateway로 전달되고, 유레카로부터 마이크로서비스의 위치 정보를 전달 받고 해당하는 정보로 직접 포워딩 시켜주는 과정

<br>

- 방법
1. eureka-client, application.yml(유레카에 등록) (Spring cloud Gateway, first, second service)
- pom.xml (Spring cloud gateway, first, second service)
```xml 
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
- application.yml (Spring cloud gateway, first, second service)
```yaml
client:
register-with-eureka: true
fetch-registry: true
service-url:
  defaultZone: http://localhost:8761/eureka
```

3. application.yml(Spring cloud gateway) : discovery server에 등록되어 있는 이름으로 포워딩
```yaml
routes:
- id: first-service
  uri: lb://MY-FIRST-SERVICE
```
3. Eureka Server 에 service 등록 확인  
   <img src="https://github.com/user-attachments/assets/ae0a8fa3-bc68-4e02-93cf-7ef73674578a" width="700" height="150">

4. http://localhost:8000/second-service/welcome => 정상 확인

5. First, Second Service 2개 기동
- port 번호 0번 , instance-id 추가 
```yaml
  instance:
    instance-id: ${spring.application.name}:${spring.application.instance_id:${random.value}}
```

```java
public class FirstServiceController {   // port 출력
    Environment env;

    @Autowired
    public FirstServiceController(Environment env) {
        this.env = env;
    }
   @GetMapping("/check")
   public String check(HttpServletRequest request) {
      log.info("Server port = {}", request.getServerPort());
      return String.format("Hi, this is a message from First Service on PORT %s",
              env.getProperty("local.server.port"));
   }
}
```
- eureka, spring cloud gateway, first service2개 , second service 2개 총6개가 기동 되었다.
- <img src="https://github.com/user-attachments/assets/171c8801-911d-4645-bfaa-8984889d4472" width="700" height="150">
- http://localhost:8000/first-service/check => 
- Hi, this is a message from First Service on PORT 54271, Hi, this is a message from First Service on PORT 54244
- 위와 같이 라우팅 기능과 로드밸러스 기능도 사용가능
