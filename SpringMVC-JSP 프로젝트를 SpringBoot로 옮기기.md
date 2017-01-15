# SpringMVC-JSP 프로젝트를 Spring Boot로 옮기기

SpringMVC 3.2.# 기반에 JSP로 되어있던 레거시 프로젝트를 Spring Boot로 전환했다.
역시나 설정 부분에서 어려운 점이 많은데 까먹기 전에 남겨두기로 한다.

## Spring Security

기존 xml로 되어 있던 설정이 Java Config에서 어떻게 매칭되는지 하나하나 파악하기가 어려워 그냥 기존의 xml 그대로 쓰기로 했다.

>MainApplication.java 에 @ImportResource({
        "classpath:/config/spring/context-security.xml", ... 를 지정해주는 걸로 해결

## static 파일

비교적 간단하다.

>webapp/static 에 있던 파일을 resources/static 으로 이동

## WebRoot(webapp)에 있던 index.html 파일

비교적 간단하다.

>webapp/index.html 을 resources/static/index.html 로 이동

## JSP

기본적으로 [jar로 만드는 SpringBoot에서는 JSP를 사용할 수 없다](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-web-applications.html#boot-features-jsp-limitations)라고 되어 있다.

하지만 방법이 없는 건 아니고, Spring Boot 1.4.3을 기준으로 해법이 나뉜다.

### Spring Boot 1.4.2 이하

해결 방법은 http://hillert.blogspot.kr/2016/03/spring-boot-with-jsp-in-executable-jar.html 에 나와있는데 정리하면,

1. application.yml에 다음과 같이 지정

``` yml
spring.mvc.view:
    prefix: /WEB-INF/jsp/
    suffix: .jsp
```

2. JSP 파일의 물리적 저장 위치

>기존의 `/WEB-INF/jsp` 에 저장했던 jsp 파일을 Spring Boot에서는 `resources/META-INF/resources/WEB-INF/jsp`에 저장해야 한다.

### Spring Boot 1.4.3 이상

1.4.3에서는 그냥 `src/main/webapp` 폴더를 만들어서 예전과 같이 `/webapp/WEB-INF/jsp`에 JSP 파일을 넣으면 된다.


## SiteMesh2

인터넷을 뒤져보면 `SiteMesh 3`과 Spring Boot를 설정하는 내용은 있어도 `SiteMesh 2`에 대한 내용은 도움이 될만한 내용이 거의 없다.

그래서 일단 `SiteMesh 3`으로 올린다. `SiteMesh 2`와 `SiteMesh 3`은 `GroupId` 마저 다르다. 암튼 Gradle dependency에 `SiteMesh 3`으로 설정 후 다시 import.

아래와 같은 기존의 `decorators.xml`의 내용을,

```xml
<?xml version="1.0" encoding="UTF-8" ?>

<decorators defaultdir="/WEB-INF/decorators">
    <excludes>
        <pattern>/user/login*</pattern>
        <pattern>/admin/login*</pattern>
        <!--<pattern>/user/signup/save</pattern>-->
    </excludes>


    <decorator name="main" page="main_decorator.jsp">
        <pattern>/admin/user*</pattern>
    </decorator>

    <decorator name="usermode" page="usermode_decorator.jsp">
        <pattern>/application/*</pattern>
        <pattern>/mail/*</pattern>
        <pattern>/payment/*</pattern>

    ... 이하 생략 ...
```

다음과 같이 ConfigurableSiteMeshFilter을 상속한 클래스에 설정하고, FilterRegistrationBean을 통해 서블릿 필터로 등록한 후,  `decorators.xml`은 삭제한다(내버려둬봐야 `decorators` 요소를 찾을 수 없다는 에러만 난다).

```java
@Bean
public FilterRegistrationBean siteMeshFilter() {
    FilterRegistrationBean filter = new FilterRegistrationBean();
    filter.setFilter(new ConfigurableSiteMeshFilter() {
        @Override
        protected void applyCustomConfiguration(SiteMeshFilterBuilder builder) {
            builder.addExcludedPath("/user/login*");
            builder.addExcludedPath("/admin/login*");
            builder.addDecoratorPath("/admin/user*", "classpath:/decorators/main_decorator.jsp");
            builder.addDecoratorPath("/admin/*", "classpath:/decorators/adminmode_decorator.jsp");
            builder.addDecoratorPath("/dbadmin/*", "classpath:/decorators/dbadminmode_decorator.jsp");
            builder.addDecoratorPath("/template/*", "classpath:/decorators/temp_decorator.jsp");
            builder.addDecoratorPath("/application/*", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/mail/*", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/payment/*", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/main/*", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/mypage/*", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/qna/*", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/pds/*", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/test/*", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/common/error.jsp", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/common/404.jsp", "classpath:/decorators/usermode_decorator.jsp");
            builder.addDecoratorPath("/common/displayTransLang.jsp", "classpath:/decorators/usermode_decorator.jsp");
        }
    });
    filter.addUrlPatterns("/*");
    return filter;
}
```

## 한글 깨짐

천신만고 끝에 애플리케이션을 띄우고 index.html을 열면 한글이 뙇!!

### 1단계 해결 방법

검색해보면 쉽게 찾을 수 있는 방법이다.

``` java
@Bean
public HttpMessageConverter<String> responseBodyConverter() {
    return new StringHttpMessageConverter(Charset.forName("UTF-8"));
}

@Bean
public Filter characterEncodingFilter() {
    CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter();
    characterEncodingFilter.setEncoding("UTF-8");
    characterEncodingFilter.setForceEncoding(true);
    return characterEncodingFilter;
}
```

그런데 이걸로 안 될때도 있더라.. 그럴땐 2단계로.

### 2단계 해결 방법

이 방법은 검색으로는 금방 안 나오는데 `application.yml`에 다음 내용을 지정하면 된다.

``` yml
spring.http.encoding:
    force: true
    force-request: true
    force-response: true
```

하는 김에 아래 내용도 추가하자. standalone Tomcat 시절에 `server.xml`에 항상 해주던 설정이다.

``` yml
server.tomcat.uri-encoding: UTF-8
```

### jar로 실행 시 한글 깨짐

이 경우는 java 실행 옵션 문제다. `JAVA_OPTS`라는 환경 변수에 `-Dfile.encoding=UTF-8`을 추가해준다.


## jar 파일 생성 후 실행 시 에러

Gradle의 `jar` 태스크를 실행해서 생성된 jar 파일을 `java -jar`로 실행하면 다음과 같은 에러가 날 수 있다.

![Imgur](http://i.imgur.com/oG4w23N.png)

이럴 때는 Gradle의 `bootRepackage` 태스크를 실행해서 jar를 만들고, `java -jar`로 실행하면 정상적으로 실행된다.

## Spring Security TagLib 관련 에러

JSP 파일이 잘 인식이 되더라도, 실제 JSP 파일을 불러보면 아래와 같은 에러가 난다.

>The absolute uri: http://www.springframework.org/security/tags cannot be resolved in either web.xml or the jar files deployed with this application

`http://www.springframework.org/security/tags`는 기본적인 `spring-boot-starter-*`에는 포함되어 있지 않으므로 수동으로 `build.gradle`에 `compile group: 'org.springframework.security', name: 'spring-security-taglibs'`을 추가해주면 된다.


## Spring Boot 설정 관련 일반적인 문제 해법

설정 관련 한다고 했는데도 원하는 대로 되지 않는다면, `localhost:8080/configprops` 로 현재 설정 정보 JSON 을 확인해보는 것이 문제 해결에 드는 시간을 줄여준다. 그리고 설정 항목은 [Spring Boot 문서](https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html#common-application-properties)에서 확인할 수 있다.

잘 해결되지 않던 한글 문제를 예로 들어보면, `localhost:8080/configprops`에서 확인해 본 결과

``` json
"spring.http.encoding-org.springframework.boot.autoconfigure.web.HttpEncodingProperties": {
    "prefix": "spring.http.encoding",
    "properties": {
        "charset": "UTF-8",
        "force": false,
        "mapping": null,
        "forceRequest": false,
        "forceResponse": false
    }
},
```

라고 되어 있어서 답을 찾을 수 있었다.


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.

