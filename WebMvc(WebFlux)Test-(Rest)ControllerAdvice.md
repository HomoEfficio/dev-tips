# @WebMvc(WebFlux)Test 와 @(Rest)ControllerAdvice

@WebMvc(WebFlux)Test는 [API 문서](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/autoconfigure/web/servlet/WebMvcTest.html)에 따르면 컨트롤러와 관련된 최소한의 애플리케이션 컨텍스트만 구성해서 테스트 구동 시간을 최소화 한다.

>Annotation that can be used for a Spring MVC test that focuses only on Spring MVC components.
Using this annotation will disable full auto-configuration and instead apply only configuration relevant to MVC tests (i.e. **@Controller, @ControllerAdvice, @JsonComponent, Converter/GenericConverter, Filter, WebMvcConfigurer and HandlerMethodArgumentResolver beans but not @Component, @Service or @Repository beans**).

>By default, tests annotated with @WebMvcTest will also auto-configure Spring Security and MockMvc (include support for HtmlUnit WebClient and Selenium WebDriver). For more fine-grained control of MockMVC the @AutoConfigureMockMvc annotation can be used.

>Typically @WebMvcTest is used in combination with @MockBean or @Import to create any collaborators required by your @Controller beans.

이렇게 보면 `@CongrollerAdvice`는 항상 빈으로 추가하는 것처럼 보이지만, `@WebMvc(WebFlux)Test(XXXController.class)`와 같이 컨트롤러를 특정했을 경우에는 `@CongrollerAdvice`가 붙은 객체를 자동으로 애플리케이션 컨텍스트에 추가해주지 않으므로, `@CongrollerAdvice` 클래스를 `@Import`로 지정해줘야 의도한대로 동작한다.

이에 대해 [이슈](https://github.com/spring-projects/spring-boot/issues/12979)가 제기된 적이 있으나 개선되지는 않은 채 종료된 듯.



