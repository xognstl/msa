### 1. Spring Cloud Netflix Eureka
___

1개의 서비스를 인스턴스가 3개로 아래의 URL과 같이 부하분산  
instance A : http://localhost:8080 , http://my-server1:8080  
instance B : http://localhost:8080 , http://my-server1:8081  
instance C : http://localhost:8080 , http://my-server1:8082

서버가 3대 이면 같은 포트사용 , 한대에서 여러서비스 일때는 포트를 다르게 해야한다.  
마이크로서비스는 스프링 클라우드 넷플릭스 유레카에 등록할 예정  
유레카가 해주는 역할은 서비스 디스커버리
- 서비스 디스커버리: 외부에서 다른 서비스들이 마이크로 서비스를 검색하기 위한 개념, 어떠한 서비스가 어디에 위치해 있는지 저장해둔다.
  
<img src="https://github.com/user-attachments/assets/59d859cf-96dc-4a34-9614-6f2f42cef517" width="700" height="400">  

클라이언트 -> API 게이트웨이 -> 서비스 디스커버리 -> 필요한 서비스의 서버정보를 반환  

SpringBootApplication 페이지에 @EnableEurekaServer 라는 어노테이션이 있다.  
새로 만드는 boot 프로젝트는 Eureka 서버 역할을 할것이다.  

<br>

### 2. Eureka Service Discovery - 프로젝트 생성
___

- 스프링 부트 프로젝트 생성
- Eureka server dependency 추가 후 프로젝트 생성
- 스프링 부트 2.4.2 버전 , spring cloud 버전 2020.0.0 사용

```yaml
server:
  port: 8761

spring:
  application:
    name: discoveryservice
    
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```
- spring.application.name : 각각 마이크로서비스 아이디
- register-with-eureka :  Eureka 서버에 자신의 정보를 등록할지 
- fetch-registry : Eureka 서버로부터 다른 서비스들의 레지스트리를 가져올지 
eureka.client 는 false 기본값은 true, 이서버는 eureka 서버라 false(true 이면 자기 자신을 가져오는거기 때문)

```java
@SpringBootApplication
@EnableEurekaServer // 어노테이션 추가 
public class DiscoveryserviceApplication {
    
}
```
- @EnableEurekaServer 추가 해준다 -> 해당 프로그램이 유레카 서버 역할을 한다.  
 => http://127.0.0.1:8761/ 접속시  
  <img src="https://github.com/user-attachments/assets/c88c51eb-c00f-48c8-8236-6b38ac4ee467" width="700" height="370">

<br>


