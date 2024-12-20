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

<br>

### 3. 비대칭키를 이용한 암호화
___

- Encryption, Decryption 할때 키가 다르다.
- Public, Private key 사용(JDK Keytool 이용)

1. 폴더 생성 후 해당 폴더에서 cmd 창에서 아래와 같은 명령어 입력
```text
keytool -genkeypair -alias apiEncryptionKey -keyalg RSA -dname "CN=xognstl, OU=API Development,
 O=msaproject, L=Seoul, C=KR" -keypass "test1234" -keystore apiEncryptionKey.jks 
 -storepass "test1234"
 ** 이어서 써야한다. "" 안은 서명 정보 
```
2. apiEncryptionKey.jks 파일이 생성 된다.
3. keytool -list -keystore apiEncryptionKey.jks -v 명령어 입력시 상세 정보 읽을 수 있다.     
=> Entry type: PrivateKeyEntry 확인 가능
4. keytool -export -alias apiEncryptionKey -keystore apiEncryptionKey.jks -rfc -file trustServer.cer 명령어 실행    
=> trust Server.cer 라는 인증서 파일 생성 됨.
5. keytool -import -alias trustServer -file trustServer.cer -keystore publicKey.jks  
=> trustServer.cer 라는 인증서로 publicKey.jks 파일 생성
6. keytool -list -keystore publicKey.jks -v => Entry type: trustedCertEntry 확인 가능  
7. publicKey.jks : public 키 , apiEncryptionKey.jks : private 키 

- config 서버에 키 정보 입력
```yaml
encrypt:
  key-store:
    location: file:///F:\msa_project\keystore\apiEncryptionKey.jks
    password: test1234
    alias: apiEncryptionKey
```

- POST 127.0.0.1:8888/encrypt / raw : xognstl -> AQAoj15guBnjSOWn8... 라고 결과 나옴
- POST 127.0.0.1:8888/decrypt / raw : AQAoj15guBnjSOWn8... -> xognstl 

- user-service.yaml , db 암호 암호화 해서 입력
```yaml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:testdb  
    username: sa
    password: '{cipher}AQAC6Y5jN0czQXAIdz.....w='
token:
  expiration_time: 86400000
  secret: user_token_natvie_application_changed_#1

gateway:
  ip: 127.0.0.1
```
- http://127.0.0.1:8888/user-service/default => 비밀번호 정상 보임
