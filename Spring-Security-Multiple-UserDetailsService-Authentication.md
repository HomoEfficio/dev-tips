# Spring Security Multiple UserDetailsService Authentication

하나의 스프링 부트 애플리케이션에서 판매자 로그인, 고객 로그인을 모두 처리할 수 있을까?

물론 있다.

그것도 여러가지 방법이 있지만 가장 간단한 방법인 다수의 UserDetailsService를 설정하는 방법을 알아보자. UserDetailsService가 DB에서 사용자 정보를 찾아서 UserDetails로 변환한 후 반환하면, UserDetails 객체는 이후 내부적으로 DaoAuthenticationProvider 에 전달되고 여기에서 비밀번호가 자동으로 검증된다.

본론으로 돌아와서 하나의 스프링 부트 애플리케이션에서 인증 과정을 여러개 사용할 수 있는 기본 아이디어는 다음과 같다.

- 판매자용 UserDetailsService 구현체, 고객용 UserDetailsService 구현체를 따로 만든다.
- 이 두 구현체를 Spring Security 에 등록해줘야 하는데 아쉽게도 하나의 Security 설정 클래스에서 다음과 같이 설정하면, 둘 중 하나만 인증 과정에 참여하고 나머지 하나는 무시된다.

    ```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(new SellerUserDetailsService(sellerRepository))
            .passwordEncoder(passwordEncoder());
        auth.userDetailsService(new CustomerUserDetailsService(customerRepository))
            .passwordEncoder(passwordEncoder());
    }
    ```

- 따라서 **Security 설정 클래스를 두 개(또는 그 이상)를 만들고, 각 클래스에서 path 를 적절히 지정해서 두 개의 UserDetailsService 구현체가 상황에 맞게 호출되게 한다.**

예를 들어, 일반적인 Security 설정에서 판매자 인증까지 담당하고, 고객 인증용 Security 설정 클래스를 따로 뺀다면 다음과 같이 작성하면 된다.

```java
@EnableWebSecurity
@RequiredArgsConstructor
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    private final SellerRepository sellerRepository;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .csrf().disable()
                .httpBasic()
                .and()
                .authorizeRequests()
                .antMatchers(HttpMethod.POST, "/v1/sellers").permitAll()  // 필요 시 상황에 맞게 /**, /* 등 추가
                .antMatchers("/h2-console/**").permitAll()
                .anyRequest().authenticated()
                .and()
                .headers()  // 아래는 h2-console 용
                .addHeaderWriter(
                        new XFrameOptionsHeaderWriter(
                                new WhiteListedAllowFromStrategy(List.of("localhost"))
                        )
                )
                .frameOptions().sameOrigin()
        ;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }

    // 판매자 인증
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(new SellerUserDetailsServiceImpl(sellerRepository)).passwordEncoder(passwordEncoder());
    }
}

```

```java
@EnableWebSecurity
@RequiredArgsConstructor
// 고객용 Security만 이 설정 클래스에서 최우선 적용하고, 나머지는 기존 Security 설정 클래스에서 처리
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CustomerSecurityConfig extends WebSecurityConfigurerAdapter {

    private final PasswordEncoder passwordEncoder;
    private final CustomerRepository customerRepository;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .csrf().disable()
                .httpBasic()
                .and()
                .requestMatchers()  // 아래 명시한 path 는 CustomerSecurityConfig에서 담당
                .antMatchers("/v1/product-reviews")  // 필요 시 상황에 맞게 /**, /* 등 추가
                .antMatchers("/v1/customers")
                .and()
                .authorizeRequests()
                .antMatchers(HttpMethod.POST, "/v1/customers").permitAll()
                .anyRequest().authenticated()
        ;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(new CustomerUserDetailsServiceImpl(customerRepository)).passwordEncoder(passwordEncoder);
    }
}

```

실행 로그를 보면 다음과 같이 `@Order(Ordered.HIGHEST_PRECEDENCE)`를 지정한 고객용 Security 설정이 먼저 적용되고 그 다음에 일반 Security 설정이 적용됨을 확인할 수 있다.

```
[           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: OrRequestMatcher [requestMatchers=[Ant [pattern='/v1/product-reviews'], Ant [pattern='/v1/customers']]], [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@58739e5e, org.springframework.security.web.context.SecurityContextPersistenceFilter@456aa471, org.springframework.security.web.header.HeaderWriterFilter@7bb4ed71, org.springframework.security.web.authentication.logout.LogoutFilter@69ba3f4e, org.springframework.security.web.authentication.www.BasicAuthenticationFilter@1657b017, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@4cfcac13, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@4276ad40, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@4e2cdc51, org.springframework.security.web.session.SessionManagementFilter@1930a804, org.springframework.security.web.access.ExceptionTranslationFilter@c732e1c, org.springframework.security.web.access.intercept.FilterSecurityInterceptor@1d944fc0]
[           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: any request, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@734a149a, org.springframework.security.web.context.SecurityContextPersistenceFilter@7fd751de, org.springframework.security.web.header.HeaderWriterFilter@2c16677c, org.springframework.security.web.authentication.logout.LogoutFilter@4a9869a8, org.springframework.security.web.authentication.www.BasicAuthenticationFilter@75e0a54c, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@e162a35, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@1124910c, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@6ce9771c, org.springframework.security.web.session.SessionManagementFilter@27d73d22, org.springframework.security.web.access.ExceptionTranslationFilter@4656fcc5, org.springframework.security.web.access.intercept.FilterSecurityInterceptor@4ced17f3]
```
