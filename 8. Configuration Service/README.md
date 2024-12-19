## 섹션 8. Configuration Service

### 1. Spring Cloud Config
___

- 분산 시스템에서 서버, 클라이언트 구성에 필요한 설정정보를 외부 시스템에서 관리
- 중앙화된 저장소에서 구성요소 관리 가능, 빌드하지 않고 적용가능

<br>

### 2. Spring Cloud Config - 프로젝트 생성
___

- F:\msa_project\git-local-repo 경로에 ecommerce.yml 파일 생성
```yaml
token:
  expiration_time: 86400000
  secret: user_token

gateway:
  ip: 127.0.0.1
```
- git init 
- git add ecommerce.yml 
- git commit -m "message"


- Config-service 프로젝트 생성
- dependency : Spring Cloud Config 의 Config Server
- @EnableConfigServer 어노테이션 추가
```java

@SpringBootApplication
@EnableConfigServer
public class ConfigServiceApplication {
}
```
```yaml
server:
  port: 8888

spring:
  application:
    name: config-service
  cloud:
    config:
      server:
        git:
          uri: F:\msa_project\git-local-repo
```

- http://127.0.0.1:8888/ecommerce/default 접속시 ecommerce.yml 파일의 내용 가져올 수 있다.
```json
{
  "name": "ecommerce",
  "profiles": [
    "default"
  ],
  "label": null,
  "version": "040403cbecbb199755610d81941ff152b0af9970",
  "state": null,
  "propertySources": [
    {
      "name": "F:\\msa_project\\git-local-repo/file:C:\\Users\\diquest\\AppData\\Local\\Temp\\config-repo-17509943428349510963\\ecommerce.yml",
      "source": {
        "token.expiration_time": 86400000,
        "token.secret": "user_token",
        "gateway.ip": "127.0.0.1"
      }
    }
  ]
}
```
<br>

### 3. Users Microservice에서 Spring Cloud Config 연동
___

- user service 에서 Spring cloud 연동
- dependency 추가 
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```
- bootstrap.yml 은 application.yml보다 우선순위가 높다.
- bootstrap.yml 파일 추가
```yaml
spring:
  cloud:
    config:
      uri: http://127.0.0.1:8888
      name: ecommerce
```
- configration server가 변경이 되면 적용 방법
- server 재기동, actuator refresh, spring cloud bus 사용
- spring boot actuator : application 상태 모니터링, metric 수집을 위한 http end point 제공
- actuator 적용방법 : dependency 추가 
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
- http.authorizeRequests().antMatchers("/actuator/**").permitAll(); => actuator는 인증없이 사용하는 코드 추가
- yaml 설정 파일에 추가
```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh, health, beans
```
- http://127.0.0.1:61818/actuator/health 
- http://127.0.0.1:61818/actuator/beans => 빈정보 확인
- POST http://127.0.0.1:61818/actuator/refresh  
- ecommerce.yml 파일 변경후 git add ecommerce.yml => git commit -m "message" 하고 refresh 를 하면 적용된다.

<br>

### 4. Spring Cloud Gateway에서 Spring Cloud Config 연동
___
1. config, bootstrap, actuator dependency 추가
2. bootstrap.yml 추가 (Config-service 정보)
3. actuator : refresh, health, beans, httptrace 추가
4. application.yml 에 User-Service 의 actuator 정보 추가

<br>

- httptrace를 사용하기 위해선 빈등록 필요

```java
@SpringBootApplication
public class ApigatewayServiceApplication {
    @Bean
    public HttpTraceRepository httpTraceRepository() {
        return new InMemoryHttpTraceRepository();
    }
}
```
-yaml 파일에 actuator route 등록
```yaml
- id: user-service
  uri: lb://USER-SERVICE
  predicates:
    - Path=/user-service/actuator/**
    - Method=GET,POST
  filters:
    - RemoveRequestHeader=Cookie
    - RewritePath=/user-service/(?<segment>.*), /$\{segment}
```

<br>

### 5. Profiles을 사용한 Configuration 적용
___

- ecommerce-dev.yml , ecommerce-prod.yml 여러가지를 사용하는 방법
- yml파일 여러개 생성 후 profiles 추가(user, gateway) 
```yaml
spring:
  cloud:
    config:
      uri: http://127.0.0.1:8888
      name: ecommerce
  profiles:
    active: prod
```
- user는 dev, gateway는 prod 로 설정하고 회원가입 -> 로그인 후 토큰으로 user-service의 health check 를 해보면  
401 에러가난다. gateway를 이용하여 user-service로 접근하기때문에 서로다르니 에러가나는것.

<br>

### 6. Remote git Repository
___

- git remote -v : 현재 remote 되있는것
- git remote add origin https://github.com/xognstl/spring-cloud-config.git
- git push --set-upstream origin master : push 처음에만 upstream 옵션 사용
- uri 변경 id/ password 도 입력 가능
```yaml
spring:
  application:
    name: config-service
  cloud:
    config:
      server:
        git:
          uri: https://github.com/xognstl/spring-cloud-config.git
          username: xognstl@naver.com
          password: password
```

<br>

### 7. Native File Repository
___

- config-service 의 yaml 파일에 profiles.active , native.search-locations 추가
```yaml
spring:
  application:
    name: config-service
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: F:\msa_project\native-file-repo
        git:
          uri: F:\msa_project\git-local-repo
```

- http://127.0.0.1:8888/ecommerce/dev, http://127.0.0.1:8888/ecommerce/default 등 테스트
- http://127.0.0.1:8888/user-service/native 
