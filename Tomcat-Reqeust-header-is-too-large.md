# Tomcat Request header is too large 오류 처리

다음과 같이 Request header is too large 에러가 나면,

![Imgur](https://i.imgur.com/WuiBNBx.png)

Tomcat 설정으로 `maxHttpHeaderSize` 값을 늘려줘야 한다.

## Embedded Tomcat

Spring Boot 에 내장된 것처럼 Embedded Tomcat 인 경우, application.yml 파일에 다음과 같이 설정. 값은 테스트 해보며 적절한 값을 지정한다.

```yml
server.max-http-header-size=20000
```

## Standalone Tomcat

서버에 Standalone 방식으로 설치된 Tomcat 인 경우, server.xml 파일에 다음과 같이 설정. 값은 테스트 해보며 적절한 값을 지정한다.

![Imgur](https://i.imgur.com/74panf0.png)

