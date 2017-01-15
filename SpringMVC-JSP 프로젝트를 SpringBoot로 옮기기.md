# SpringMVC-JSP 프로젝트를 Spring Boot로 옮기기

SpringMVC 3.2.# 기반에 JSP로 되어있던 레거시 프로젝트를 springboot로 전환했다.
역시나 설정 부분에서 어려운 점이 많은데 까먹기 전에 남겨두기로 한다.

## Spring Security

기존 xml로 되어 있던 설정이 Java Config에서 어떻게 매칭되는지 하나하나 파악하기가 어려워 그냥 기존의 xml을 그대로 쓰기로 했다.

>MainApplication.java 에 @ImportResource({
        "classpath:/config/spring/context-security.xml", 로 해결

## JSP

결론적으로 [jar로 만드는 SpringBoot에서는 JSP를 사용할 수 없다](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-web-applications.html#boot-features-jsp-limitations).

해결 방법은 http://hillert.blogspot.kr/2016/03/spring-boot-with-jsp-in-executable-jar.html 에 나와있는데, 정리하면,

>JSP를 다음의 링크에서 설명하는 것처럼 기존의 /WEB-INF/jsp 에 저장했던 jsp 파일을 Spring Boot에서는 resources/META-INF/resources/WEB-INF/jsp 에 저장해야 한다.

## sitemesh2

인터넷을 뒤져봐도 SiteMesh 3과 Spring Boot를 설정하는 내용은 있어도 SiteMesh2 에 대한 내용은 없다.

그래서 고전 중..

