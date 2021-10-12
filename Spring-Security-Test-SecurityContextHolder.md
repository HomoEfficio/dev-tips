# Spring Security Test SecurityContextHolder

Spring Security 테스트 시 시큐리티 정보를 mock 하는 방법은 여러가지가 있다.

https://docs.spring.io/spring-security/site/docs/current/reference/html5/#test

authentication 에 커스텀 객체를 저장하고 이를 SecurityContextHolder 에 넣어서 사용하고 있다면, 테스트 시 다음과 같이 SecurityContextHolder 에 커스텀 객체를 넣어서 테스트 할 수 있다.

```kotlin
    @BeforeEach
    internal fun beforeEach() {
        val context = SecurityContextHolder.createEmptyContext()
        val mockAuthentication = TestingAuthenticationToken(
            myCustomObj, // principals 자리에 커스텀 객체 저장,
            null,  // credentials 은 상황에 따라 null 도 가능
        )
        context.authentication = mockAuthentication
        SecurityContextHolder.setContext(context)
    }
```

