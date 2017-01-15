# SpringMVC-JSP 프로젝트를 Spring Boot로 옮기기

SpringMVC 3.2.# 기반에 JSP로 되어있던 레거시 프로젝트를 springboot로 전환했다.
역시나 설정 부분에서 어려운 점이 많은데 까먹기 전에 남겨두기로 한다.

## Spring Security

기존 xml로 되어 있던 설정이 Java Config에서 어떻게 매칭되는지 하나하나 파악하기가 어려워 그냥 기존의 xml 그대로 쓰기로 했다.

>MainApplication.java 에 @ImportResource({
        "classpath:/config/spring/context-security.xml", 로 해결

## static 파일

비교적 간단하다.

>webapp/static 에 있던 파일을 resources/static 으로 이동

## WebRoot에 있던 index.html 파일

비교적 간단하다.

>webapp/index.html 을 resources/static/index.html 로 이동

## JSP

결론적으로 [jar로 만드는 SpringBoot에서는 JSP를 사용할 수 없다](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-web-applications.html#boot-features-jsp-limitations).

해결 방법은 http://hillert.blogspot.kr/2016/03/spring-boot-with-jsp-in-executable-jar.html 에 나와있는데, 정리하면,

>JSP를 다음의 링크에서 설명하는 것처럼 기존의 /WEB-INF/jsp 에 저장했던 jsp 파일을 Spring Boot에서는 resources/META-INF/resources/WEB-INF/jsp 에 저장해야 한다.

## sitemesh2

인터넷을 뒤져봐도 SiteMesh 3과 Spring Boot를 설정하는 내용은 있어도 SiteMesh2 에 대한 내용은 없다.

그래서 일단 SiteMesh 3으로 올린다. SiteMesh 2와 3은 Group Id 마저 다르다. 암튼 Gradle dependency에 SiteMesh 3으로 설정 후 reimport.

아래와 같은 기존의 decorators.xml은 삭제하고,

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

다음과 같이 ConfigurableSiteMeshFilter을 상속한 클래스에 설정하고, FilterRegistrationBean을 통해 서블릿 필터로 등록한다.

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


