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

<br>

### 2. Prometheus와 Grafana
___
- Prometheus
  - Metrics 를 수집하고 모니터링 및 알람에 사용되는 오픈소스 애플리케이션
  - Time Series Database(TSDB) 사용 => 각종 지표가 시간순으로 정보가 남는다.
  - Pull 방식의 구조와 다양한 Metric Exporter 제공
  - 시계열 DB에 Metrics 저장 -> 조회 가능 (Query)

- Grafana
    - 데이터 시각화, 모니터링 및 분석을 위한 오픈소스 애플리케이션
    - 시계열 데이터를 시각화하기 위한 대시보드 제공

- 프로메테우스에서 스프링 클라우드가 수집했던 정보를 가져와 저장한다. 그 정보를 가지고 그라파나 시각화 시켜준다.

- prometheus 다운로드 https://prometheus.io/download/
- yaml 파일에 prometheus 다운 받은곳이랑 지정해줘야함.
- prometheus.exe 파일로 서버 실행
  - .\prometheus.exe
  - ./prometheus --config.file=prometheus.yml
- 127.0.0.1:9090 Prometheus Dashboard

- Grafana 다운로드 https://grafana.com/get/?tab=self-managed 
- .\bin\grafana-server.exe server 실행
- ./bin/garafana-server
- http:127.0.0.1:3000 admin/admin 
- prometheus 와 연동

- apigateway 에 order-service 의 actuator 정보 추가
```yaml
- id: order-service
  uri: lb://ORDER-SERVICE
  predicates:
    - Path=/order-service/actuator/**
    - Method=GET
  filters:
    - RemoveRequestHeader=Cookie
    - RewritePath=/order-service/(?<segment>.*), /$\{segment}
```
- prometheus.yml
```yaml
static_configs:
  - targets: ["localhost:9090"]
- job_name: 'user-service'
  scrape_interval: 15s
  metrics_path: '/user-service/actuator/prometheus'
  static_configs:
    - targets: ['localhost:8000']
- job_name: 'apigateway-service'
  scrape_interval: 15s
  metrics_path: '/actuator/prometheus'
  static_configs:
    - targets: ['localhost:8000']
- job_name: 'order-service'
  scrape_interval: 15s
  metrics_path: '/order-service/actuator/prometheus'
  static_configs:
    - targets: ['localhost:8000'] 
```
- prometheus dashboard 에 http_server_requests_seconds_count 를 검색하면 관련정보와 시각화 그래프 확인 가능

- grafana -> prometheus 연동
- 왼쪽 connections -> data sources -> add data source -> prometheus -> prometheus url 추가 
- grafana dashboard 에서 Import dashboard 선택
- https://grafana.com/grafana/dashboards/
- micrometer JVM , Prometheus 2.0 Overview , Spring cloud Gateway -> dashboard id copied
- import dashboard 에 카피한 아이디 붙히고 prometheus 지정 후 import

- 그라파나에서 보이지 않는 부분은 http://127.0.0.1:8000/user-service/actuator/metrics을 참고하여
- sum(spring_cloud_gateway_requests_seconds_count{job=~"$gatewayService"}) 이런 식으로 바꿔준다.
