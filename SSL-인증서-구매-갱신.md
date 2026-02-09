# SSL 인증서 구매 갱신

## 큰 흐름

- 만료 예정 기존 인증서 확인
- 인증서 구매
- 이메일로 인증서와 설치 가이드 수령
- 도메인 소유 여부 심사
- 인증서 수령 및 확인
- 실제 서버 인증서 설치

## 만료 예정 기존 인증서 확인

- 브라우저의 주소창을 통해 인증서 pem 파일 다운로드
  - <img width="568" height="663" alt="ssl-01" src="https://github.com/user-attachments/assets/fbcfdd58-4d56-4241-af4f-d787e567671e" />
- 다음 명령으로 인증서 확인(여기에서는 github.com 인증서 샘플)
  - `openssl x509 -in github.com.pem -noout -text`
  - 명령 결과의
    - Issuer > CN 값으로 인증서 발급자확인 가능(ex: CN=Digicert)
    - Subject > CN 값으로 와일드카드 인증서 여부 확인 가능(ex: CN=*.abc.com 이면 와일드카드 인증서)
   
## 인증서 구매

- 인증서를 구매하려면 CSR(Certificate Signing Request) 파일이 필요하며,
- CSR 파일을 생성하려면 개인키 파일이 있어야 한다.

### 개인키 파일 생성

- openssl 명령으로 생성
  - `openssl genrsa -des3 2048 > encrypted_key.pem`
  - 비밀번호 입력

### 개인키 파일 복호화

- CSR 파일 생성 및 실제 서버에 인증서를 설치할 때는 복호화 된 개인키 파일이 필요하므로 다음 명령으로 미리 생성해둔다.
  - `openssl rsa -in encrypted_key.pem -out key.pem`
  - 비밀번호 입력

### CSR 파일 생성 및 구매 요청

- 다음 명령으로 CSR 파일 생성
  - `openssl req -new -key key.pem -out csr.pem`
  - 와일드카드 인증서 CSR 입력 내용 sample
    - Country Name: KR
    - State or Province Name (full name) [Some-State]: Gyeonggi-do
    - Locality Name: Seongnam-si
    - Organization Name: My Best Company
    - Organization Unit Name: 그냥 엔터
    - Common Name: *.abc.com
    - Email Address: 그냥 엔터
    - A challenge password: 그냥 엔터
    - An optional company name: 그냥 엔터
- 인증서 판매사 요청 양식에 맞게 요청 및 CSR 파일 첨부

## 도메인 소유 여부 심사

- 대개 인증서 판매사로부터 구매하고자 하는 인증서를 적용할 도메인의 실제 소유자인지 확인하는 절차를 거친다.
- 보통 판매사가 공유해준 문자열을 DNS 서버의 TXT 레코드로 추가하도록 요청한다.(즉 TXT 레코드를 추가할 수 있으면 실제 소유자로 간주)
- DNS 서버에 TXT 레코드를 등록한 후 [Google DNS 조회 도구](https://toolbox.googleapps.com/apps/dig/?lang=ko#TXT/) 나 [WhatsMyDNS](https://www.whatsmydns.net/dns-lookup/txt-records) 에서 확인 가능
- 판매사 쪽에서 TXT 레코드를 확인한 후 실제 인증서를 이메일로 보내준다.

## 인증서 수령 및 확인

### 인증서 파일 확인

- 인증서 메일에는 다음 파일이 포함돼 있다.
  - CSR 파일
  - 인증서 파일(ex: cert.pem)
  - 체인 인증서인 경우 다음 파일이 추가로 포함된다.
    - 중간 인증서 파일(ex: DigiCertCA.pem)
    - 루트 인증서 파일(ex: TrustedRoot.pem)
- 체인 인증서 관련 일반적인 정보는 https://github.com/HomoEfficio/dev-tips/blob/master/TLS-SSL-Certificate-Chain-of-Trust.md 를 참고한다.
- 받은 인증서가 정상 동작하는지 로컬 nginx 에 설치해서 미리 확인한다.

### 체인 인증서 생성

- 체인 인증서인 경우 다음과 같이 인증서, 체인인증서, 루트인증서의 순서대로 하나의 pem 파일로 합친다.
  - cat cert.pem DigiCertCA.pem TrustedRoot.pem > cert_chained.pem

### 인증서 로컬 확인

- 로컬에 nginx 를 설치하고
- 다음과 같이 nginx.conf 파일의 HTTPS 서버 블록 주석을 해제하고 다음과 같이 체인 인증서를 설치한다.
  ```
    # HTTPS server
    #
    server {
        listen       443 ssl;
        server_name  localhost;

        ssl_certificate      FULL_PATH_TO_CHAINED_CERT.pem;
        ssl_certificate_key  FULL_PATH_TO_DECRYPTED_KEY.pem;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
  ```
- `nginx -t` 명령으로 conf 파일 오류 없음을 확인하고
- `nginx` 명령으로 nginx 서버를 실행한다.
- `openssl s_client -connect localhost:443 | openssl x509 -noout -dates` 명령으로 새 인증서의 만료일을 확인할 수 있다.
- 브라우저의 주소창을 통해서도 확인 가능하다.
- `curl -vI https://localhost:443` 으로도 확인 가능하다.

## 실제 서버 인증서 설치

- 실제 서버에 설치하는 방법은 환경에 따라 다르나, 결국 (체인) 인증서와 복호화 된 개인키 파일만 있으면 인증서를 설치할 수 있다.
  - 직접 설치: conf 파일에 (체인) 인증서, 복호화 된 개인키 파일 위치 지정
  - 클라우드 환경: 보통 인증서 관리 기능이 있으며 (체인) 인증서와 복호화 된 개인키 파일을 등록하고 사용할 수 있다.
  - CDN: 보통 인증서 관리 기능이 있으며 마찬가지로 (체인) 인증서와 복호화 된 개인키 파일을 등록하고 사용할 수 있다.
  - k8s: 보통 Secret에 (체인) 인증서와 복호화 된 개인키 파일을 등록하고 사용할 수 있다.


