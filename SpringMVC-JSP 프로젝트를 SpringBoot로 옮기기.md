# SpringMVC-JSP 프로젝트를 Spring Boot로 옮기기

SpringMVC 3.2.# 기반에 JSP-SiteMesh2로 되어있던 레거시 프로젝트를 Spring Boot로 전환했다.

역시나 설정 부분에서 어려운 점이 많은데 까먹기 전에 남겨두기로 한다.


## Spring Boot 1.4.2

Spring Boot에서 

>- JSP와 SiteMesh를 사용하면서
>
>- FAT JAR로 배포하고 싶다면
>
>- **Spring Boot 1.4.2를 사용**해라.

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

>기존의 `/WEB-INF/jsp` 에 저장했던 jsp 파일을 Spring Boot에서는 **`resources/META-INF/resources/WEB-INF/jsp`에 저장**해야 한다.

그리고 아래 `jar 파일 생성 후 실행 시 에러`에서 설명한 것처럼 `bootRepackage` task로 jar를 만들어야 제대로 실행된다.

이렇게 하면 jar 파일로는 애플리케이션이 잘 동작하는데, IDE에서 애플리케이션을 실행시키면 `/WEB-INF/`를 못찾아서 site 화면이 제대로 뜨지 않는다. IDE에서 제대로 실행되지 않는다면 debug 모드를 못 쓴다는 의미이므로 사실상 개발이 어렵다.

편법이지만 webapp 폴더가 jar 생성 시 무시된다는 성질을 역이용하면 이 문제를 해결할 수 있다.

3. webapp 폴더 활용

>`resources/META-INF/resources/WEB-INF/` 폴더 내의 내용을 `src/main/webapp/WEB-INF/` 폴더 내로 복사한다.

이렇게 하면 IDE에서 실행할 때도 `/WEB-INF/`를 문제 없이 찾으므로 debug 모드를 활용할 수 있고,
webapp 폴더는 jar에는 포함되지 않으므로 배포에도 영향이 없다.


### Spring Boot 1.4.3 이상

`resources/META-INF/resources/WEB-INF/`와 `webapp/WEB-INF/`를 병행 사용하는 전략이 먹히지 않는다.

1.4.3을 쓰려면 Fat JAR를 포기하고 webapp 폴더를 쓰는 WAR를 선택하거나, 아니면 JSP-SiteMesh를 포기해야 한다.


## jar 파일 생성 후 실행 시 에러

Gradle의 `jar` 태스크를 실행해서 생성된 jar 파일을 `java -jar`로 실행하면 다음과 같은 에러가 날 수 있다.

![Imgur](http://i.imgur.com/oG4w23N.png)

이럴 때는 다음과 같이 해결한다.

>Gradle의 `bootRepackage` 태스크를 실행해서 jar를 만들고, 
>
>`java -jar`로 실행하면 정상적으로 실행된다.



## Spring Security TagLib 관련 에러

JSP 파일이 잘 인식이 되더라도, 실제 JSP 파일을 불러보면 아래와 같은 에러가 날 수 있다.

>The absolute uri: http://www.springframework.org/security/tags cannot be resolved in either web.xml or the jar files deployed with this application

이 오류의 발생 원인은 **`http://www.springframework.org/security/tags`는 기본적인 `spring-boot-starter-*`에는 포함되어 있지 않기 때문**이다. 따라서 다음과 같이 의존 관계를 추가해주면 된다.

>**`build.gradle`에 `compile group: 'org.springframework.security', name: 'spring-security-taglibs'`을 추가**해주면 된다.


## Spring Security

기존 xml로 되어 있던 설정이 Java Config에서 어떻게 매칭되는지 하나하나 파악하는데 시간이 많이 소요되어 그냥 기존의 xml 그대로 쓰기로 했다.

>MainApplication.java 에 @ImportResource({
        "classpath:/config/spring/context-security.xml", ... 를 지정해주는 걸로 해결


## static 파일

비교적 간단하다.

>webapp/static 에 있던 파일을 resources/static 으로 이동


## WebRoot(webapp)에 있던 index.html 파일

비교적 간단하다.

>webapp/index.html 을 resources/static/index.html 로 이동


## SiteMesh2

Spring Boot로의 전환에서 가장 속 썩는 부분이 바로 SiteMesh 설정이다.

인터넷을 뒤져보면 `SiteMesh 3`과 Spring Boot를 설정하는 내용은 있어도 `SiteMesh 2`에 대한 내용은 도움이 될만한 내용이 거의 없다.

기존에는 `SiteMesh 2`를 사용해서 decorator도 JSP로 만들었는데, `SiteMesh 3`은 일단 홈페이지의 예제가 JSP가 아니라 HTML로만 되어 있어서, 3으로 올리면 소스 수준에서 decorator를 모두 수정해야 할 판이다. 그래서 그냥 `SiteMesh 2`를 그대로 사용하기로 했다.

해결 방법은 다음과 같다.

>기존에 `src/main/webapp/WEB-INF/decorators.xml`파일은 **`src/main/resources/META-INF/resources/WEB-INF/decorators.xml`로 옮기고**,
>
>`SiteMeshConfig`라는 `@Configration` 클래스로 **`SiteMeshFilter`만 `Bean`으로 등록**해주면 다행스럽게도 잘 동작한다.

``` java
import com.opensymphony.sitemesh.webapp.SiteMeshFilter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SiteMeshConfig {

    @Bean
    public FilterRegistrationBean siteMeshFilter() {

        FilterRegistrationBean filter = new FilterRegistrationBean();
        filter.setFilter(new SiteMeshFilter());
        return filter;
    }
}
```

>`src/main/resources/META-INF/resources/WEB-INF/decorators.xml`에서 `<decorators defaultdir="/WEB-INF/decorators">`로 지정했으므로, **decorator 파일들은 `src/main/resources/META-INF/resources/WEB-INF/decorators` 폴더 내부에 위치**한다.

참고로 jsp로 된 decorator 파일은 대략 아래와 같은 형식으로 시작한다.

``` jsp
<%@ page contentType="text/html;charset=UTF-8"%>
<%@ include file="/WEB-INF/jsp/common/env.jsp"%> // <-- include 할 jsp 파일의 위치를 /WEB-INF/jsp/~~로 지정
<%@ taglib uri="http://www.opensymphony.com/sitemesh/decorator" prefix="decorator"%>
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

확실하게는 **`java -Dfile.encoding=UTF-8 -jar`로 jar 파일을 실행**하면 된다.


## context.getResourceAsStream(strPath)

Spring 1.4.2 라면 읽어오고 싶은 자원을 **`src/main/resources/META-INF/resources/`를 `/`로 해서 strPath를 기술**하면 된다. 

예를 들어 `src/main/resources/META-INF/resources/WEB-INF/abc.txt` 파일은 `context.getResourceAsStream("/WEB-INF/abc.txt")`로 InputStream에 담을 수 있다.

하지만 `src/main/resources/static/def.txt` 파일을 `context.getResourceAsStream("/def.txt")`로 읽을 수는 없다.


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

라고 되어 있어서, 이를 발판 삼아 답을 찾을 수 있었다.


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.

