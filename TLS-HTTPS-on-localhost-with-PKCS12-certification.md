# TLS HTTPS on localhost with PKCS12 certification

PCKS12 사설 인증서를 생성해서 localhost 에 TLS(HTTPS)를 적용해보자.

### 사설 인증서 생성

아래와 같이 localhost 에서 사용할 수 있는 인증서 생성 (비번은 monolith로 입력)

```
🍺  keytool -genkeypair -alias localhost -keyalg RSA -keysize 2048 -storetype PKCS12 -keystore monolith.p12 -validity 3650
키 저장소 비밀번호 입력:  
새 비밀번호 다시 입력: 
이름과 성을 입력하십시오.
  [Unknown]:  Homo Efficio
조직 단위 이름을 입력하십시오.
  [Unknown]:  
조직 이름을 입력하십시오.
  [Unknown]:  homo_efficio.io
구/군/시 이름을 입력하십시오?
  [Unknown]:  
시/도 이름을 입력하십시오.
  [Unknown]:  
이 조직의 두 자리 국가 코드를 입력하십시오.
  [Unknown]:  82
CN=Homo Efficio, OU=Unknown, O=homo_efficio.io, L=Unknown, ST=Unknown, C=82이(가) 맞습니까?
  [아니오]:  y
```
- 위 명령을 실행한 폴더에 monolith.p12 파일 생성됨
- src/main/resources/keystore 폴더로 이동

### HTTPS 설정

- application.yml

```yml
server.port: 8443

server.ssl:
  key-store-type: PKCS12
  key-store: classpath:keystore/monolith.p12
  key-store-password: monolith
  key-alias: localhost
```

### HTTPS 적용된 서버 시작

- 서버 시작 로그에 아래와 같이 `(https)`가 표시됨

    ![Imgur](https://i.imgur.com/FQA1vCg.png)
    
- localhost:8080 으로는 아예 접근할 수 없으며, localhost:8443 으로 접근하면 아래와 같이 TLS 요구

    ![Imgur](https://i.imgur.com/3H6Vm5y.png)
    
- https://localhost:8443 으로 접근하면 비공인 사설 인증서라서 아래와 같은 경고 표시됨

    ![Imgur](https://i.imgur.com/uClEozC.png)
    
- 고급 버튼을 클릭하면 다음과 같이 안전하지 않은 localhost 로 이동하겠냐고 묻는다.

    ![Imgur](https://i.imgur.com/d5oo6Sf.png)

- `localhost로 이동`을 클릭하면 다음과 같이 서버에 연결된다.

    ![Imgur](https://i.imgur.com/SKWBHhg.png)

- 느낌표를 클릭하면 다음과 같이 인증서가 표시된다.

    ![Imgur](https://i.imgur.com/wYYfLUu.png)
    
- 인증서를 클릭하면 다음과 같이 인증서 정보를 확인할 수 있다.

    ![Imgur](https://i.imgur.com/cJvlg3Q.png)

# 기타

사설 인증서를 사용하면 요청을 날리는 도구에 따라 다음과 같은 에러가 발생할 수 있음

```
x509: certificate is not valid for any names, but wanted to match localhost"
또는
x509: cannot validate certificate for 127.0.0.1 because it doesn't contain any IP SANs"
또는
x509: certificate signed by unknown authority"
```

이때는 도구에서 제공하는 인증서 검증 우회 옵션을 사용해야 한다. 예를 들어 k6 는 아래와 같이 `--insecure-skip-tls-verify` 옵션을 사용해서 실행하면 된다.

>k6 run --insecure-skip-tls-verify <<<K6-script.js>>>
