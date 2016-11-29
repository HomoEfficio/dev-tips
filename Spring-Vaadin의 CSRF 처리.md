# Spring-Vaadin의 CSRF 처리

Vaadin에서는 자체적으로 CSRF(Cross Site Request Forgery) 처리를 하고 있기 때문에 Spring Security에서 CSRF 처리를 해주면  Vaadin에 접속하지 못하게 될 수도 있다.

따라서 Spring Security 설정 부분에서 `csrf().disable()`와 같이 아예 CSRF 처리를 비활성화 시켜버리라고 권고하고 있는데, 그럼에도 불구하고 일부 end-point에 대해 CSRF 처리가 필요하다면 어떻게 해야할까?

## csrf().ignoringAntMatchers()로 분기

Spring Security 설정을 담당하는 클래스에서 다음과 같이 `csrf().ignoringAntMatchers()`로 분기해주면, CSRF 방지 처리를 적용하는 경우와 안 하는 경우를 request url 차원에서 나눠서 처리할 수 있게 된다.

```java
@Configuration
@EnableWebSecurity
@Order(SecurityProperties.ACCESS_OVERRIDE_ORDER)
public class PocApiSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    public void configure(HttpSecurity httpSecurity) throws Exception {
        httpSecurity                
//              .csrf().disable()  // 이렇게 하면 CSRF 전체가 비활성화
                // 아래와 같이 CSRF 처리를 해주지 않을 URL만 따로 지정
                .csrf().ignoringAntMatchers("/api1/**", "/api2/**", "/api3/**")
                .and()
                .authorizeRequests()
                .antMatchers("/abc/**").permitAll()
                .antMatchers("/def/**").hasRole("USER")                
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .and()
                .logout();
    }
    
    ...
}    
```

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
