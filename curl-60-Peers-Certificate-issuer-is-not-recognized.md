# curl: (60) Peer's Certificate issuer is not recognized.

TLS 인증서가 설치 된 서버(omw.com)를 대상으로 리눅스에서 `curl -v https://omw.com`와 같이 curl을 해보면 다음과 같은 에러가 발생할 때가 있다.

```
curl -v https://omw.com    // omw.com 은 실제 존재하는 호스트가 아닌 그냥 예시
*   Trying 222.222.222.222:443...
* Connected to abc.com (222.222.222.222) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*  CAfile: none
*  CApath: none
* loaded libnssckbi.so
* ALPN: offers h2,http/1.1
* Server certificate:
* subject: 어쩌고저쩌고
* NSS error -8179 (SEC_ERROR_UNKNOWN_ISSUER)
* Peer's Certificate issuer is not recognized.
* Closing connection 0
curl: (60) Peer's Certificate issuer is not recognized.
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

안내에 https://curl.se/docs/sslcerts.html 를 보라는데 봐도 그닥 해결책은 보이지 않는다.

그런데 동일한 서버를 대상으로 맥에서 `curl -v https://omw.com` 해보면 아무 문제 없이 정상 실행된다.

맥의 curl은 7.79.1 이다.

그래서 CentOS에 있던 curl을 기존 7.29.0 에서 최신 버전인 8.0.1로 업그레이드를 했는데([여기](https://hongddo.tistory.com/182) 참고), 그래도 마찬가지 에러가 발생한다.

검색하다보니 답은 [여기](https://serverfault.com/questions/1102356/curl-peers-certificate-issuer-is-not-recognized-error-when-attempting-to-comm#comment1438840_1102356)에서 찾았다.

대략 정리해보면 원인은 RedHat이나 CentOS 리눅스에서 사용하는 curl은 NSS를 사용하는데 이게 `/etc/pki/tls/certs/`에 있는 것들을 사용하지 않기 때문이라고 한다.

실제로 CentOS에서 `curl -V`를 실행해보면 다음과 같이 `NSS`라는 놈이 보인다.

```
curl -V
curl 8.0.1 (x86_64-redhat-linux-gnu) libcurl/8.0.1 NSS/3.79 zlib/1.2.7 libpsl/0.20.2 (+libidn2/2.3.2) libssh2/1.10.0 nghttp2/1.33.0
Release-Date: 2023-03-20
Protocols: dict file ftp ftps gopher gophers http https imap imaps ldap ldaps mqtt pop3 pop3s rtsp scp sftp smb smbs smtp smtps telnet tftp
Features: alt-svc AsynchDNS GSS-API HSTS HTTP2 HTTPS-proxy IPv6 Kerberos Largefile libz NTLM NTLM_WB PSL SPNEGO SSL UnixSockets
```

그래서 NSS가 사용되는 curl에서도 인증서를 인식할 수 있도록 `update-ca-trust`라는 도구가 제공된다고 한다.

`update-ca-trust` 관련 자세한 내용은 https://zetawiki.com/wiki/CentOS_update-ca-trust 를 참고한다.


## 해결

### `update-ca-trust` 실행

인증서 파일을 `/etc/pki/ca-trust/source/anchors/` 디렉터리에 복사하고 `sudo update-ca-trust`를 실행한 후에 다시 curl 을 해보면 인증서를 인식하고 정상 실행된다. 인증서 파일은 보통 브라우저의 자물쇠를 클릭하고 아래 그림과 같이 내보내기를 통해 다운로드 할 수 있다.

![Imgur](https://i.imgur.com/8EGq0Hn.png)

이렇게 해도 안 된다면 curl을 업그레이드 해야 한다. CentOS curl 7.29.0 (x86_64-redhat-linux-gnu) 에서는 `update-ca-trust`를 실행해도 계속 Peer's Certificate issuer is not recognized. 에러가 발생하며 curl을 8.0.1로 업그레이드 하면 정상 실행 된다.
