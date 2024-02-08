# curl --resolve

개발/운영하다보면 DNS에 등록된 IP를 새로운 IP로 변경을 해야할 때가 있는데,
이 때 호스트가 새로운 IP를 통해 의도한대로 동작하는지 미리 확인할 수 있으면 좋다.

이럴 때 hosts 파일 내용을 수정해서 새로운 IP로 연결되게 하는데 더 간편한 방법이 있었다.

예를 들어 my-server.example.com의 IP를 새로운 IP인 33.44.55.66으로 변경하려고 할 때,
다음과 같이 curl에 --resolve 옵션을 사용해서 일종의 dry-run 처럼 미리 확인할 수 있다.

```
curl https://my-server.example.com  --resolve my-server.example.com:443:33.44.55.66 -vvv

* Added my-server.example.com:443:33.44.55.66 to DNS cache
* Hostname my-server.example.com was found in DNS cache
*   Trying 33.44.55.66:443...
* Connected to my-server.example.com (33.44.55.66) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (OUT), TLS handshake, Client hello (1):
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-AES256-GCM-SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: 생략
*  start date: Feb 23 00:00:00 2023 GMT
*  expire date: Mar 20 23:59:59 2024 GMT
*  subjectAltName: host "my-server.example.com" matched cert's "생략"
*  issuer: 생략
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x15980ca00)
> GET / HTTP/2
> Host: my-server.example.com
> user-agent: curl/7.79.1
> accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 404
< date: Wed, 07 Feb 2024 05:47:07 GMT
< content-type: application/problem+json
< content-length: 107
< vary: Origin
< vary: Access-Control-Request-Method
< vary: Access-Control-Request-Headers
< cache-control: no-cache, no-store, max-age=0, must-revalidate
< pragma: no-cache
< expires: 0
< x-content-type-options: nosniff
< strict-transport-security: max-age=15724800; includeSubDomains
< x-xss-protection: 1 ; mode=block
< referrer-policy: no-referrer
< set-cookie: SCOUTER=z2165e00ko10o9; Path=/; Max-Age=2147483647; Expires=Mon, 25 Feb 2092 09:01:14 GMT
<
* Connection #0 to host my-server.example.com left intact
생략%
```
