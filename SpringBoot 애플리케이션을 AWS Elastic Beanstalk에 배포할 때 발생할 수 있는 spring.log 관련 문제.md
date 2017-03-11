# SpringBoot 애플리케이션을 AWS Elastic Beanstalk에 배포할 때 발생할 수 있는 spring.log 관련 문제

logback을 사용하는 SpringBoot 애플리케이션을 AWS Elastic Beanstalk에 배포하면 다음과 같이 `spring.log` 파일에 대한 권한이 없다면서 정상 기동하지 않는 경우가 있다.

- 에러 로그 추가

## 원인

`logback.xml`을 보니 아래와 같은 내용이 포함되어 있었다.

```xml
<include resource="org/springframework/boot/logging/logback/base.xml"/>
```

그래서 [base.xml](https://github.com/spring-projects/spring-boot/blob/master/spring-boot/src/main/resources/org/springframework/boot/logging/logback/base.xml) 파일을 찾아보니 오류의 원인이 었던 `spring.log` 관련 내용이 포함되어 있다.

```xml
<property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}}/spring.log}"/>
```

`logback.xml`의 정확한 문법을 모르지만 대략 `LOG_FILE`, `LOG_PATH`, `LOG_TEMP`, `java.io.tmpdir`의 값을 A라 하면 `A/spring.log` 파일에, A가 없다면 `/tmp/spring.log` 파일에 로그를 남긴다는 얘기 같다.

`sudo find / -name 'spring.log'`로 찾아보면 몇몇 `spring.log` 파일이 나오는데, 모두 권한은 `tomcat`으로 되어 있어서 권한 문제는 없을 것 같은데 암튼 실제로는 권한 문제로 에러가 나고 있다. 

## 해결

파일을 아예 지우고 다시 기동시켜봐도 마찬가지 오류가 나고, `LOG_FILE` 같은 속성의 값을 특정 폴더로 지정해줘도 오류가 난다. 

그래서 좀 무식하지만 아예 `base.xml`을 include 하는 대신에 아래와 같이 `base.xml`의 내용을 `logback.xml` 파일에 인라인화 시켜버리고, 로그 파일 위치를 `LOG_FILE`로 직접 지정했더니 오류가 나지 않았다.

```xml
<include resource="org/springframework/boot/logging/logback/defaults.xml" />
<property name="LOG_FILE" value="/var/log/tomcat8/spring.log}"/>
<include resource="org/springframework/boot/logging/logback/console-appender.xml" />
<include resource="org/springframework/boot/logging/logback/file-appender.xml" />
<root level="INFO">
    <appender-ref ref="CONSOLE" />
    <appender-ref ref="FILE" />
</root>
```

## logback 로깅 설정 파일에 프로파일 적용

위와 같이 하면 Elastic Beanstalk 환경에서는 잘 도는데 로컬 개발 환경에서 문제가 생길 수도 있다. 따라서 로깅 설정에도 프로파일을 적용하는 것이 좋겠다.

로깅 파일에 프로파일을 적용하려면 [스프링 부트 공식 문서](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-logging.html#boot-features-logback-extensions)에 따라 먼저 `logback.xml` 파일의 이름을 `logback-spring.xml` 로 바꿔야 한다. 

그리고나서 다음과 같이 `<springProfile>`을 써서 프로파일을 적용할 수 있다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="production">
        <include resource="org/springframework/boot/logging/logback/defaults.xml" />
        <property name="LOG_FILE" value="/var/log/tomcat8/spring.log}"/>
        <include resource="org/springframework/boot/logging/logback/console-appender.xml" />
        <include resource="org/springframework/boot/logging/logback/file-appender.xml" />
        <root level="INFO">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="FILE" />
        </root>
        
        <logger name="~~~" level="~~~" />
        ...
    </springProfile>
    <springProfile name="local">
        <include resource="org/springframework/boot/logging/logback/base.xml"/>

        <logger name="~~~" level="~~~" />
        ...
    </springProfile>
</configuration>
```



----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
