# TLS(SSL) Certificate Chain of Trust

HTTPS 웹 사이트를 방문하고 브라우저 주소창에서 '자물쇠 아이콘 > 이 연결은 안전합니다 > 인증서가 유효함'을 차례로 클릭하면 다음과 같이 인증서 뷰어를 통해 인증서 내용을 확인할 수 있다.

세부정보 탭을 클릭하면 맨 위에 '인증서 계층'이라고 나온다.

![Imgur](https://i.imgur.com/UfpGDt5.png)

계층을 선택하고 '내보내기'를 클릭하면 계층별 인증서를 파일로 각각 내려받을 수 있고, 파일을 텍스트 에디터로 열어보면 아래와 같이 생겼다.

```
-----BEGIN CERTIFICATE-----
어쩌구저쩌구...
-----END CERTIFICATE-----
```

그런데 인증서 계층이란 무엇일까?


## 인증서 계층

사실 TLS 인증서를 검색해보면 계층 구조에 대한 설명 없이 CA(Certificate Authority)와 End-entity(쉽게 말해 그냥 인증서를 요청하고 발급받은 조직/개인) 사이에서 비대칭키를 사용해서 정보 누설 없이 안전하게 통신할 수 있는 메커니즘을 소개하는 자료들이 많이 나온다.

아마도 쉽게 설명하기 위해 그렇게 했겠지만, 현재 사용되는 대부분의 TLS 인증서는 `ROOT CA 인증서 - Intermediate CA 인증서 - End-entity 인증서` 이렇게 최소 3계층 구조를 가지고 있다. 이 계층 구조를 Chain of Trust라고 부르기도 한다.

그렇다면 왜 Intermediate CA(ICA) 인증서를 중간에 두는 최소 3계층 구조(ICA 인증서를 하나 더 두면 4계층이 된다)를 사용하는 걸까?

완전한 설명은 아니겠지만 쉽게 설명해보면 다음과 같다.


### 위험 분산

예를 들어 A가 인증서 발급 기관 B의 비밀키를 탈취했다면, A는 악의적인 사이트 C를 만들고, 마치 B가 발급한 것처럼 사이트 C에 대한 인증서 D를 만들 수 있다. 사이트 C 접속자(의 브라우저)는 인증서 D를 믿고 사이트에 방문해서 정보를 입력한다. A는 그 정보를 악용할 수 있다. 결국 인증서 발급 기관의 비밀키가 탈취되면 보안이 유지되지 않는다.

이런 상황에서 극소수의 ROOT CA(RCA)가 절대 다수의 End-entity(EE)에 대한 인증서를 직접 발급하는 상황에서 RCA의 비밀키가 탈취되면 굉장히 많은 수의 EE에 대한 보안이 깨지게 된다. 그래서 발급 기관의 수를 늘리면 하나의 발급 기관당 발급하는 EE 인증서의 수는 줄어들 게 되므로 위험 분산 효과가 있다.

그런데 이런 논리라면 그냥 RCA를 많이 만들면 될 뿐이므로 최소 3계층 구조가 꼭 필요하다는 근거로는 빈약하다.


### ROOT CAs 난립 방지

RCA에 대한 인증서는 누가 발급해주는 걸까? 놀랍게도 RCA 스스로 발급한다. 진짜?

![Imgur](https://i.imgur.com/gfOTFOH.png)

그렇다. 쉽게 말해 RCA는 최상위 ROOT에서 '그냥 무조건 나를 믿어~'라고 말하는 존재다.
하지만 무조건 믿을 수야 있나. RCA는 외부 감사를 받아야 하고, 브라우저나 OS 제조사의 심사 대상이 되며, 정보 투명성 유지 의무가 있어서 이를 통해 커뮤니티도 수시로 악의적인 행동이나 보안 위험 노출 여부를 감시할 수 있다. 그리고 RCA로서 부적절하다면 RCA 박탈(revocation) 메커니즘이 동작해서 브라우저나 OS가 해당 RCA의 인증서를 제거해서 피해 확산을 막게 된다.

이런 RCA가 난립하게 둔다면 굉장히 비효율적인 구조가 될 수 밖에 없으므로 RCA의 수는 적절히 제한하는 것이 유리하다. RCA의 수는 적절히 제한하되 발급 기관의 수를 늘려서 위험 분산을 하려면 어떻게 해야할까? 바로 이 지점에서 계층 구조가 필요하다.


### 효율성과 유연성

계층 구조를 도입하면 '소수의 RCA - 다수의 ICA - 절대 다수의 EE' 구조가 만들어진다. 그래서 RCA는 2계층 구조에 비해 3계층 구조일 때 더 적은 수의 인증서를 발급하게 된다. 이는 하나의 RCA가 감시해야 할 대상(인증서를 발급받은 대상)이 줄어든다는 것을 의미한다.

인증 기관은 인증서를 발급하고 끝나는 것이 아니라 발급 대상이 악의적인 행동을 하거나 보안 위험에 노출되지 않는지 감시해야 하며, 부적절한 경우 발급 대상에 대한 인증서를 취소해서 피해 확산을 막는 능동적인 역할을 담당한다. 따라서 발급 대상 감시와 인증서 취소에 대해 다양하고 유연한 전략을 구사할 수 있다면 인증서를 사용하는 체계 전체의 보안성을 높일 수 있게 된다. 다계층 구조에서는 ICA가 저마다 유연한 전략을 구사할 수 있으므로 보안성을 높일 수 있다.


## 인증서 설치

앞서 대부분의 인증서가 최소 3계층 구조를 가지고 있다는 사실을 확인해봤고, 왜 그런 구조를 가져야 하는지 살펴봤다.

이제 인증서가 서버에 어떻게 설치되는지 간단하게 알아보자.

앞서 살펴본 것처럼 대부분의 인증서는 RCA인증서(셀프 발급)-ICA인증서(RCA가 발급)-EE인증서(ICA가 발급) 이렇게 최소 3계층으로 구성된다.

서버에 인증서를 설치할 때는 3계층의 인증서를 모두 포함하는 하나의 파일로 설치해야 한다. 구체적으로는 다음과 같이 구성된다.

```
-----BEGIN CERTIFICATE-----
EE 인증서
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
ICA 인증서
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
RCA 인증서
-----END CERTIFICATE-----
```

이처럼 RCA인증서-ICA인증서-RCA인증서가 연결된 인증서를 Chained Certificate이라고 부른다. 맨 위에 EE인증서가 있고 맨 아래에 RCA인증서가 있는 순서에 유의하자.

서버가 저 Chained Certificate 파일을 인지할 수 있어야 하며, 예를 들어 nginx의 경우 nginx.conf 파일에 다음과 같이 지정해서 인증서를 설치한다.

```
server {
    listen       443  ssl;
    server_name  your.server.name;

    ssl_certificate  ChainedCertificate파일경로;
    ssl_certificate_key  비밀키파일경로;
```

## 인증서 설치 오류

만약 올바른 Chained Certificate 파일이 아니라 아래와 같이 EE인증서만 포함된 파일을 사용하면 어떻게 될까?

```
-----BEGIN CERTIFICATE-----
EE 인증서
-----END CERTIFICATE-----
```

이렇게 하면 골때리는 현상이 발생한다.

일부 애플리케이션에서는 해당 서버에 HTTPS 연결이 정상적으로 맺어지는데, 일부 애플리케이션에서는 연결이 되지 않는다.


### Java 애플리케이션 사례

예를 들어 Java 애플리케이션을 사용해서 연결이 실패할 때는 대략 다음과 같은 에러 메시지가 나온다.

```
javax.net.ssl.SSLHandshakeException: General OpenSslEngine problem
...
Caused by: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
...
Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
...
```

### curl 사례

curl을 사용해서 연결이 실패할 때는 대략 다음과 같은 에러 메시지가 나온다.

```
curl -v https://your.server.name/
*   Trying xxx.xxx.xxx.xxx...
* TCP_NODELAY set
* Connected to your.server.name (xxx.xxx.xxx.xxx) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS alert, Server hello (2):
* SSL certificate problem: unable to get local issuer certificate
* stopped the pause stream!
* Closing connection 0
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Java나 curl에서 알려주는 메시지는 너무 구체성이 떨어져서 인증서 관련 뭔가 문제가 있구나 정도만 알 수 있을 뿐 정확히 무슨 문제인지는 알기 어렵다. 어떻게 해야할까?


## 인증서 문제 확인 및 해결

인증서 문제를 더 구체적으로 확인할 수 있는 온라인 서비스들이 있다. `SSL Server Test`로 검색하면 여럿 나온다.

그중에서 간단하고 빠르게 동작하는 https://www.digicert.com/help/ 를 예로 살펴보자.

Server Address란에 your.server.name을 입력하고 Check Server를 클릭하면 다음과 같이 ICA 인증서가 없다는 사실을 바로 알려준다.

![Imgur](https://i.imgur.com/H8ZAmZ0.png)

위에서 살펴본 것처럼 ICA 인증서와 RCA 인증서를 차례로 EE 인증서 아래에 붙여주면 Chained Certificate 파일이 된다.  
서버를 재시작하면 올바른 Chained Certificate 파일이 인식되며, 다시 digicert 사이트에서 확인해보면 다음과 같이 정상임을 확인할 수 있다.

![Imgur](https://i.imgur.com/mxIWABf.png)

ICA 인증서와 RCA 인증서는 아마도 회사내 어디엔가 있겠지만, 정 찾기 어렵다면 이 글 맨 위에 있는 것처럼 인증서 보기의 내보내기 기능을 통해 ICA 인증서와 RCA 인증서를 내려받아 사용하면 된다.

참고로 https://www.ssllabs.com/ssltest/ 를 이용하면 훨씬 자세한 진단 결과를 받을 수 있다.

