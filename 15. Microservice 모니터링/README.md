## 섹션 15. Microservice 모니터링

### 1. Micrometer
___

- 각종 마이크로서비스에서 발생하는 상황, 성능, 모니터링을 위해 Hystrix, turbine 서버 같은것을 사용 했었다.
- 현재는 Hystrix Dashboard / Turbine => Micrometer + Monitoring System 사용


- Micrometer
  - JVM 기반의 애플리케이션의 Metrics 제공
  - Spring Framework 5, Spring Boot 2 부터 Spring 의 Metrics 처리
  - Prometheus 등의 다양한 모니터링 시스템 지원

- Timer
  - 짧은 지연 시간, 이벤트의 사용 빈도 측정
  - 시계열로 이벤트의 시간, 호출 빈도 등을 제공
  - @Timed 제공


- 마크가 되어있는 status와 welcome 메소드를 사용자가 호출하게 되면 그 정보가 micrometer에서 기록되고 prometheus 에서 사용된다.

#### Micrometer 적용
- user, order, api-gateway 에 dependency 추가
- yaml 에 info, metrics, prometheus 추가
```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh, health, beans, busrefresh, info, metrics, prometheus
```

- @Timed 추가
```java
@RestController
@RequestMapping("/")
public class UserController {
    @GetMapping("/health_check")
    @Timed(value = "users.status", longTask = true)
    public String status() {
        return String.format("It's Working in User Service"
                +", port(local.server.port) = " + env.getProperty("local.server.port")
                +", port(server.port) = " + env.getProperty("server.port")
                +", token secret = " + env.getProperty("token.secret")
                +", token expiration time = " + env.getProperty("token.expiration_time")
                );
    }

    @GetMapping("/welcome")
    @Timed(value = "users.welcome", longTask = true)
    public String welcome() {
        return greeting.getMessage();
    }

}
```
- welcome, status 메소드 실행 후 127.0.0.1:52705/actuator/metrics 를 호출하면 user.status, users.welcome을 확인 할 수 있다.
- 127.0.0.1:52705/actuator/prometheus 호출한 것에 대한 정보가 있다.
