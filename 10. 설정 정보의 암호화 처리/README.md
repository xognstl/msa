## 섹션 10. 설정 정보의 암호화 처리

### 1. 대칭키와 비대칭키 
___

- 대칭 암호화(symmetric Encryption) : Encryption 에 사용 되었던 키와 Decryption 에 사용되는 키를 같은 것을 쓰는경우
- 비대칭 암호화(Asymmetric Encryption) : 키를 다른 것을 쓰는것. Private and Public Key, Java Keytool 이용
  
<br>

### 2. 대칭키를 이용한 암호화
___
- Config Server bootstrap dependency 추가 , 이게 있어야 bootstrap.yml 파일 읽어온다.
```yaml
encrypt:
  key: abcdefghdsaflkjqweiasdk1234567890 #임의의 아무글자
```
- POST 127.0.0.1:8888/encrypt row data text 로 입력 xognstl -> 0b32324fc525734bd2505e8ce77cafe001bee1fcae7015efd5710f63895d383c
- POST 127.0.0.1:8888/decrypt row data text 로 입력 0b32324fc525734bd2505e8ce77cafe001bee1fcae7015efd5710f63895d383c -> xognstl

<br>

- user-service 의 db정보 옮기기 
- config server 에 native 로 db정보 옮김 password꼭 ''로 감싸야함 아니면 500에러남
- F:\msa_project\native-file-repo\user-service.yml
```yaml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:testdb  
    username: sa
    password: '{cipher}9f221c2a693b9be51722c2dbab02bc79f03863ff7fe5702eb342960417c6c636'
token:
  expiration_time: 86400000
  secret: user_token_natvie_application_changed_#1

gateway:
  ip: 127.0.0.1
```
- user-service 의 bootstrap.yml 에 user-service.yml 읽어올거라 변경 해주기
```yaml
spring:
  cloud:
    config:
      uri: http://127.0.0.1:8888
      name: user-service
```
- http://127.0.0.1:8888/user-service/native => "spring.datasource.password": "1234" 확인 가능
- password 에 다른 값을 넣으면 "invalid.spring.datasource.password": "<n/a>" 처럼 나온다.
