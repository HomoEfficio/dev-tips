# Spring Boot `@WebFluxTest` 주의사항

Spring WebFlux 는 Spring WebMvc의 Reactive 판이라고 볼 수 있다.

그래서 `@WebFluxTest`도 `@WebMvcTest`의 Reactive 판이고, 테스트에 사용할 Reactive 방식의 client 나 server 만 다를 뿐 기본적인 흐름은 같다.

라고 생각했는데 실제로는 그렇지가 않더라능..

>Spring Security 가 추가돼 있다면,  
>- `@WebMvcTest` 테스트를 실행하면 Security 설정 컨텍스트가 반영이 된 상태의 테스트 서버 인스턴스가 기동된다.
>- 그런데 **`@WebFluxTest` 테스트를 실행하면 Security 설정 컨텍스트가 반영이 안된 상태의 테스트 서버 인스턴스가 기동된다.**

이 사실을 모르고 테스트 하면 Spring Security 설정에서 분명히 `csrf().disable()`을 해줬는데도 불구하고 아래와 같이 CSRF 토큰 관련 403 에러가 계속된다.

```
java.lang.AssertionError: Status expected:<200 OK> but was:<403 FORBIDDEN>

> POST /v1/sellers
> WebTestClient-Request-Id: [2]
> Accept: [application/json]
> Content-Type: [application/json]
> Content-Length: [135]

{"id":null,"name":"HomoEfficio","email":"homo.efficio@gmail.com","phone":"010-8888-9999","loginId":"homoefficio","password":"password"}

< 403 FORBIDDEN Forbidden
< Content-Type: [text/plain]
< Cache-Control: [no-cache, no-store, max-age=0, must-revalidate]
< Pragma: [no-cache]
< Expires: [0]
< X-Content-Type-Options: [nosniff]
< X-Frame-Options: [DENY]
< X-XSS-Protection: [1 ; mode=block]
< Referrer-Policy: [no-referrer]

CSRF Token has been associated to this client

```

그래서 찾아보니 https://github.com/spring-projects/spring-boot/issues/16088 여기에 관련 내용이 있다.  
대략 `@WebFluxTest`에서는 Spring Security 가 기본으로 반영되지 않은 테스트 서버 인스턴스가 뜨는 게 맞고, 이를 스프링 부트 2.1 에 반영한다는 것 같은데, 내가 테스트한 환경은 스프링 부트 2.3 인데도 여전히 Spring Security 가 반영되지 않는다.

그러니 Spring Boot 2.3 에서 Spring Security가 반영된 `@WebFluxTest`를 수행하려면 다음과 같이 해줘야 한다.

```java
@WebFluxTest
@Import(WebFluxSecurityConfig.class)  // Spring Security 설정을 명시적으로 Import
@MockBean({SellerRepository.class, DatabaseClient.class})  // WebFluxSecurityConfig에서 주입받게 되어 있는 것들
public class SecurityTest {
    ...
```
