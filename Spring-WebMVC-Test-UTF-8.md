# Spring WebMVC Test UTF-8

스프링 WebMVC 테스트하다보면 아래와 같이 Request 나 Response 한글이 깨져 나오기도 한다.

```
MockHttpServletResponse:
           Status = 400
    Error message = null
          Headers = [Content-Type:"application/json", X-Content-Type-Options:"nosniff", X-XSS-Protection:"1; mode=block", Cache-Control:"no-cache, no-store, m
     Content type = application/json
             Body = {"message":"ì¼ë¨ ìë¬","requestUrl":"http://localhost/api/...","errors":[{"field":"description","value":"Xì¼ì´ì¼ì¬ì¤ì...
    Forwarded URL = null
   Redirected URL = null
          Cookies = []

```

스프링 부트 2.4.5 현재 application.yml 파일에 아래와 같이 설정해주면 깨지지 않는다.

```yml
server:
  servlet:
    encoding:
      charset: utf-8
      force: true
```

예전 버전이라면 아래와 같이 설정하면 되지만, deprecated 라고 나오면 위와 같이 설정하면 된다.

```yml
spring:
  http:
    encoding:
      charset: utf-8
      force: true
```
