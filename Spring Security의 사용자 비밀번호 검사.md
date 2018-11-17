# Spring Security의 사용자 비밀번호 검사

스프링 시큐리티 예제를 보며 개발하다 보면 `UserDetailsService` 인터페이스의 `loadUserByUsername(String username)`을 구현해서 사용자 정보를 DB에서 조회하고 반환한다.

하지만 비밀번호를 체크하는 코드는 없다. 그래도 잘못된 비밀번호를 입력하면 로그인에 실패한다.

내가 작성한 코드에는 없지만 어디에선가 비밀 번호 체크를 하고 있는 것이다. 비밀 번호는 어디에서 체크할까?

## DaoAuthenticationProvider

컨트롤러에서 `AuthenticationManager.authenticate(Authentication)`을 호출하면 스프링 시큐리티에 내장된 AuthenticationProvider의 `authenticate()` 메서드가 호출되는데, 이 중에서 `DaoAuthenticationProvider.additionalAuthenticationChekcs(UserDetails, UsernamePasswordAuthenticationToken)` 메서드에 다음과 같은 코드가 있다.

```java
    String presentedPassword = authentication.getCredentials().toString();

		if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
			logger.debug("Authentication failed: password does not match stored value");

			throw new BadCredentialsException(messages.getMessage(
					"AbstractUserDetailsAuthenticationProvider.badCredentials",
					"Bad credentials"));
		}
```

