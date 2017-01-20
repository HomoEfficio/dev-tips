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
>
>- 참고: https://dzone.com/articles/spring-boot-with-jsps-in-executable-jars-1

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


## MyBatis

SpringMVC 에서는 Mapper.xml 파일이 src/main/java 아래에 있어도 classpath에 포함이 되었지만, Spring Boot에서는 포함되지 않아서 실행 시 클래스패스에서 Mapper.xml 파일을 찾지 못한다.

>Mapper.xml 파일은 src/main/resources/sqlmapper 아래에 폴더별로 모아두고,
>
>Databaseconfig.java에서 다음과 같이 Mapper의 위치를 지정

``` java
@Bean("sqlSessionFactory")
public SqlSessionFactory sqlSessionFactoryBean(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
    sqlSessionFactoryBean.setDataSource(dataSource);
    sqlSessionFactoryBean.setConfigLocation(applicationContext.getResource("classpath:/config/mybatis/mybatis-config.xml"));
    sqlSessionFactoryBean.setMapperLocations(applicationContext.getResources("classpath:/sqlmapper/**/*Mapper.xml"));  // <-- 이렇게 지정
    return sqlSessionFactoryBean.getObject();
}
```


## Uncaught SyntaxError: Unexpected token o in JSON at position 1

SpringMVC에서는 Response의 content-type이 application/json이 아닌 경우, javascript에서 JSON.parse(result)를 해줘야 result 내의 데이터에 접근할 수 있었는데,

Spring Boot에서는 Response의 content-type이 `application/json`으로 넘어오므로, `JSON.parse(result)`를 실행하면 `Uncaught SyntaxError: Unexpected token o in JSON at position 1`라는 에러가 발생한다. 이유는 이미 result가 이미 JSON 객체라서 `JSON.parse("[object Object]")`와 같이 해석되어 object의 o에서 에러가 발생한다.

~~따라서 다음과 같이 `JSON.parse()`를 벗겨낸다.~~

~~var parsedData = JSON.parse(result); 를~~

~~var parsedData = result; 로 수정한다.~~

단순히 벗겨내기만 하면 `data`의 내부에 `[{...}, {...}, {...}]`와 같은 배열이 있는 경우 파싱이 안 될 수 있으므로 다음과 같이 앞에 `JSON.stringify()`를 앞에 추가해준다.

성능 관점에서 올바르지는 않지만 레거시라는 점을 감안하면 작업 관점에서는 더 효율적일 수 있다.

```java
result = JSON.stringify(result);  // 이렇게 다시 문자열화.. 전혀 아름답진 않다..
var parsedData = JSON.parse(result);
```


## The code of method _jspService(HttpServletRequest, HttpServletResponse) is exceeding the 65535 bytes limit

크기가 큰 JSP의 경우 Servlet Java 파일로 전환되면 doService() 메서드 하나에 JSP의 내용이 대부분 들어가서 `code_length`가 64k를 넘을 수가 있으며 이는 JVM 스펙에 위배되어 발생하는 에러다.

web.xml이 있던 시절에는 톰캣인 경우 아래와 같이 설정해주면 에러가 나지 않게 할 수 있었다.

```java
<servlet>
    <servlet-name>jsp</servlet-name>
    <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
    <init-param>
        <param-name>mappedfile</param-name>
        <param-value>false</param-value>
    </init-param>
</servlet>
```

Spring Boot에서는 `JspServlet`이라는 클래스를 제공해주고 있으며, `EmbeddedServletContainerCustomizer`를 통해 임베디드 서블릿 컨테이너를 커스터마이징 할 수 있으므로 다음과 같이 설정하면 된다.

```java
import org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer;
import org.springframework.boot.context.embedded.JspServlet;
//... 이하 import 생략 ...

@Configuration
public class WebConfig {

    @Bean
    public EmbeddedServletContainerCustomizer customizer() {
        return container -> {
            JspServlet jspServlet = new JspServlet();
            HashMap<String, String> initParams = new HashMap<>();
            initParams.put("mappedfile", "false");
            jspServlet.setInitParameters(initParams);
            container.setJspServlet(jspServlet);
        };

        // Java 8 미만이라면 다음과 같이 해준다.
//        return new EmbeddedServletContainerCustomizer() {
//            @Override
//            public void customize(ConfigurableEmbeddedServletContainer container) {
//                JspServlet jspServlet = new JspServlet();
//                HashMap<String, String> initParams = new HashMap<>();
//                initParams.put("mappedfile", "false");
//                jspServlet.setInitParameters(initParams);
//                container.setJspServlet(jspServlet);
//            }
//        };
    }    
}


```

## 파일 업로드 관련

### CommonsMultipartResolver vs StandardServletMultipartResolver

파일 업로드 처리에 CommonsMultipartResolver를 사용하면 request에 file이 담겨있지 않을 수 있다.

>참고: (참고: http://stackoverflow.com/questions/37951569/multipartfile-is-null-when-i-use-commonsmultipartresolver-in-my-spring-boot-app)

`CommonsMultipartResolver` 대신에 Spring 3.1부터 추가된 `StandardServletMultipartResolver`를 사용하면 정상적으로 file이 담겨 있는 request를 받을 수 있는데, Spring Boot에서는 MultipartResolover 관련 별다른 설정이 없다면 기본으로 `StandardServletMultipartResolver`를 사용한다.

따라서, `CommonsMultipartResolver`를 별도로 설정하고 있었다면 해당 빈 설정 내용을 제거한다.

### DefaultMultipartHttpServletRequest vs StandardServletMultipartResolver

`DefaultMultipartHttpServletRequest`도 마찬가지로 request에 client가 보낸 정보가 담겨있지 않은 채로 넘어올 수 있다. 

Spring Boot에서는 Spring 3.1부터 추가된 `StandardServletMultipartResolver`을 사용하므로, `DefaultMultipartHttpServletRequest` 대신 `StandardServletMultipartResolver`를 사용한다.


## 캐쉬

Spring Boot에서는 `starter-cache`로 캐쉬도 자동 설정을 지원한다.

EhCache 2.*를 사용하고 있었다면 CacheManager 관련 별도의 Bean을 설정할 필요는 없고, 두 가지만 설정해주면 된다. 

1. MainApplication에 `@EnableCaching`를 추가

2. `application.yml`에서 `ehcache.xml` 파일의 위치 지정

    >spring.cache.ehcache.config: classpath:config/ehcache/ehcache.xml


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.

