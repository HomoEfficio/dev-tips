# 파일 업로드 관련 주요 설정

## nginx

### client_max_body_size

nginx 가 받아들일 수 있는 최대 요청 본문 크기

이 크기를 넘어서는 요청이 들어오면 HTTP 413 Request Entity Too Large 를 반환한다.


## Spring Boot

https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#application-properties

### spring.servlet.multipart.file-size-threshold

멀티파트 파일이 디스크에 저장되기 시작하는 크기, 기본값 0

?? 이걸 5MB로 지정하면 5MB 씩 디스크에 써지므로 메모리를 절약할 수 있는 건가?


### spring.servlet.multipart.max-file-size

멀티파트로 들어올 수 있는 파일 1개의 최대 크기, 기본값 1MB

이를 넘어서는 파일이 들어오면 FileSizeLimitExceededException 발생


### spring.servlet.multipart.max-request-size

멀티파트로 들어올 수 있는 파일을 포함한 요청 1개의 최대 크기, 기본값 10MB

예를 들어, 510MB 로 지정돼 있다면 100MB 짜리 파일을 5개까지 하나의 요청으로 보낼 수 있다.

이를 넘어서는 요청이 들어오면 SizeLimitExceededException 발생


### server.jetty.max-http-form-post-size

HTTP POST 로 전송되는 form 데이터 최대 크기, 기본값 200000B


### server.netty.max-chunk-size

HTTP 요청 디코딩 될 수 있는 최대 크기, 기본값 8KB


### server.tomcat.max-http-form-post-size

HTTP POST 로 전송되는 form 데이터 최대 크기, 기본값 2MB


### server.tomcat.max-swallow-size

>The maximum number of request body bytes (excluding transfer encoding overhead) that will be swallowed by Tomcat for an aborted upload. An aborted upload is when Tomcat knows that the request body is going to be ignored but the client still sends it. If Tomcat does not swallow the body the client is unlikely to see the response. If not specified the default of 2097152 (2 megabytes) will be used. A value of less than zero indicates that no limit should be enforced. - https://tomcat.apache.org/tomcat-8.0-doc/config/http.html

>톰캣이 받아들일 수 있는 최대 요청 본문 바이트 수. 이 수치를 넘어서는 요청 본문은 클라이언트가 계속 보내더라도 톰캣이 무시하며, 클라이언트는 정상적인 응답을 받을 수 없다. 기본값은 2MB 이고 음수를 지정하면 무제한이다.

라고 돼있지만 실제로는 2MB로 설정을 하고 300MB가 넘는 파일을 업로드 해도 이 설정으로 인한 에러가 발생하지는 않는다.


### server.undertow.max-http-post-size

HTTP POST 로 전송되는 form 데이터 최대 크기, 기본값 -1(무제한)



