# Spring Security와 h2-console 함께 쓰기

로컬 개발 환경에서 자주 사용하는 H2 Database는 웹 클라이언트를 제공하고 있고, Spring Boot에서도 지원한다.

그래서 프로젝트 Dependency에 `spring-boot-devtools`를 포함하면 `localhost:8080/h2-console`를 통해 H2 웹 클라이언트에 접근할 수 있다.

![Imgur](http://i.imgur.com/Puj3XrY.png)

![Imgur](http://i.imgur.com/8G0SztT.png)

하지만 H2가 Spring Security를 만나면 한없이 약해지는데, `h2-console`에 접근하려면 몇 가지 조치가 필요하다.


## 1. 인증 면제 처리

Spring Security를 적용하면서 인증 면제 목록에 `h2-console`을 추가해주지 않으면, `localhost:8080/h2-console`에 접근할 때마다 디폴트 로그인창으로 자동으로 리다이렉트 되므로, H2 Console에 접근할 수 없게 된다. 

따라서 아래와 같이 `h2-console`을 Spring Security 설정 클래스의 `permitAll()`로 인증 면제 처리를 해줘야 한다.

```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                .authorizeRequests()
                    .antMatchers(
                            "/h2-console/**"    // 여기!
                    ).permitAll()
                    .anyRequest().authenticated()
                .and()
                .formLogin()
                    .loginPage("어쩌구")
                    .defaultSuccessUrl("저쩌구")
                    .permitAll()
                .and()
                .logout()
                    .logoutSuccessUrl("얼씨구")
                    .permitAll();
    }
}
```

자, 이제 `localhost:8080/h2-console`에 접근하면 `h2-console` 로그인 화면을 볼 수 있다. 하지만 로그인을 해보면 아래와 같은 화면을 만나게 된다.

![Imgur](http://i.imgur.com/3uhDO3c.png)


## 2. CSRF 면제 처리

Spring Security에서는 Cross Site Request Forgery(CSRF)를 방지 장치가 기본으로 탑재되어 있다. 

하지만 H2 Console의 로그인 화면에는 CSRF 처리가 되어 있지 않으므로 위와 같은 에러를 만나게 된다. 그렇다고 h2-console을 사용하기 위해 CSRF를 완전히 disable 하는 것은 애플리케이션 전체의 보안 수준이 낮아지므로 좋은 방법이 아니다. 

대신에 `h2-console`에 대해서만 CSRF 면제 처리를 해주는 것이 좋다.

```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                .authorizeRequests()
                    .antMatchers(
                            "/h2-console/**"
                    ).permitAll()
                    .anyRequest().authenticated()
                .and()
                .csrf()
                    .ignoringAntMatchers("/h2-console/**")    // 여기!
                .and()
                .formLogin()
                    .loginPage("어쩌구")
                    .defaultSuccessUrl("저쩌구")
                    .permitAll()
                .and()
                .logout()
                    .logoutSuccessUrl("얼씨구")
                    .permitAll();
    }
}
```

자 이제 다시 로그인을 시도해보면 되겠지?

![Imgur](http://i.imgur.com/rz365yJ.png)

뭐지 이 새하얀 순백은.. 머리 속까정 새하얘지네..


## 3. X-Frame-Options 면제 처리

정신을 차리고 개발자 도구를 열어보면 다음과 같은 에러가 나온다.

![Imgur](http://i.imgur.com/GnSDE7y.png)

네트워크 탭을 열어서 헤더의 내용을 보면,

![Imgur](http://i.imgur.com/Url87wp.png)

`X-Frame-Options`의 값이 `DENY`로 되어 있고, 왼쪽에 `heaer.jsp`, `tables.do`, `query.jsp`, `help.jsp` 등이 빨간색으로 표시되어 나온다.

`X-Frame-Options`에 sameOrigin으로 인식되 해주면 더이상 에러가 발생하지 않는다.

```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                .authorizeRequests()
                    .antMatchers(
                            "/h2-console/**"
                    ).permitAll()
                    .anyRequest().authenticated()
                .and()
                .csrf()
                    .ignoringAntMatchers("/h2-console/**")
                .and().headers().frameOptions().sameOrigin()
                .and()
                .formLogin()
                    .loginPage("어쩌구")
                    .defaultSuccessUrl("저쩌구")
                    .permitAll()
                .and()
                .logout()
                    .logoutSuccessUrl("얼씨구")
                    .permitAll();
    }
}
```

이제 다시 `localhost:8080/h2-console`에 접속해서 로그인 하면 반가운 `h2-console` 화면을 만날 수 있다. 조회 등 동작도 정상이다.
