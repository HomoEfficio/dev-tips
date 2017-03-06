# SpringBoot-Tomcat8 XSS Filter 적용

XSS(Cross Site Scripting) 공격에 대해서는 영문 자료이긴 하지만 [여기](https://excess-xss.com/)를 보면 그림만 봐도 얼추 감이 온다.

최근에는 브라우저에서도 XSS 공격 우려가 있는 스크립트의 실행을 막는 장치가 되어 있기는 하다.

![Imgur](http://i.imgur.com/1bglC5V.png)

하지만 여전히 이런 문제는 서버에서 처리해주는 것이 좋다.

XSS 공격을 막는 대표적인 방법으로는 [Lucy XSS Servlet Filter](https://github.com/naver/lucy-xss-servlet-filter)를 사용하는 것이다. 상당히 세부적인 설정과 커스터마이징을 할 수 있는 고맙고 훌륭한 라이브러리다.

한편 좀더 간단한 해결 방법도 있는데 `X-XSS-Protection`이라는 HTTP Header를 이용하는 방식이다.
Response Header에 `X-XSS-Protection: 1; mode=block`이라는 헤더를 추가해주면, 브라우저가 XSS 공격에 사용될 수 있는 스크립트를 실행하지 않는다.

이를 SpringBoot + Tomcat8에 적용할 수 있는 코드는 다음과 같다.

```java
@Bean
public FilterRegistrationBean httpHeaderSecurityFilter() {
    FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
    HttpHeaderSecurityFilter httpHeaderSecurityFilter = new HttpHeaderSecurityFilter();
    httpHeaderSecurityFilter.setXssProtectionEnabled(true);
    filterRegistrationBean.setFilter(httpHeaderSecurityFilter);
    return filterRegistrationBean;
}
``` 

Tomcat8에서는 `org.apache.catalina.filters.HttpHeaderSecurityFilter`라는 클래스를 제공해주는데, 이 클래스에 HTTP Header를 통해 보안을 강화할 수 있는 방법이 담겨 있다. 세부 사항은 https://tomcat.apache.org/tomcat-8.0-doc/config/filter.html 의 **HTTP Header Security Filter**를 참고한다.

참고로 Spring의 `FilterRegistrationBean`는 한 개의 Bean 뿐 아니라 여러 개의 Bean을 만들 수 있다.

```java
@Bean
public FilterRegistrationBean httpHeaderSecurityFilter() {
    // HttpHeaderSecurityFilter 설정
}

@Bean
public FilterRegistrationBean corsFilter() {
    // corsFilter 설정
}
```

위와 같이 `org.apache.catalina.filters.HttpHeaderSecurityFilter`를 사용하면  Response Header에 다음과 같이 `X-XSS-Protection: 1; mode=block`가 포함되고, 브라우저는 XSS 공격 우려가 있는 스크립트를 실행하지 않는다.

![Imgur](http://i.imgur.com/0IAn7yU.png)


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
