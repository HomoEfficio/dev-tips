# Spring Boot MockMvc 로 Filter 테스트

요새는 웹 애플리케이션 개발 시 서버 쪽 컴포넌트를 Mock으로 대체하는 레이어별 테스트보다는 ~~클라이언트를 모사하는~~(이 잘못된 예상이 비극을 불러왔..) MockMvc를 활용하는 통합 테스트 위주로 개발한다.

이유는 Mock 관련 코드 작성하다 진 다 빠지고, 그렇게 작성했다 해도 운영하면서 발생하는 이런저런 변경 사항에 영향도 많이 받기 때문이다. 실행하는 데 조금 더 시간이 걸리지만 그래도 조금 긴 기간동안 두고 보면 레이어별 테스트보다는 그냥 통합 테스트가 가성비가 좋다고 본다. 하지만 언제나 통합 테스트가 낫다는 건 아니며 레이어별 테스트가 더 적합한 경우도 물론 많다.

암튼 MockMvc 로 테스트하다가 정말 한참을 뻘짓을 한 것이 있으니 바로 Filter를 대상으로 하는 테스트다.

Security Filter를 추가할 일이 있어서 늘 했던 것처럼,

- `WebSecurityConfigurerAdapter`를 상속받는 설정 클래스에서 Security 관련 Filter를 추가하고,
- Filter 코드에  Breakpoint를 걸어두고 애플리케이션을 디버그 모드로 실행한 후,
- MockMvc 로 테스트를 해보면 아무리해도 Filter 내의 Breakpoint에서 멈추지 않았다.

Filter가 제대로 등록이 안 된 건가 하고, https://www.baeldung.com/spring-security-registered-filters 를 참고해서 Filter 목록을 찍어보면 등록 자체는 정상적으로 돼 있다. Filter 등록에 문제가 전혀 없는데도 Filter가 동작은 하지 않는 상황..

테스트 관련 [레퍼펀스 문서](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-testing-spring-boot-applications)를 보니 `@SpringBootTest`의 webEnvironment를 지정 안 해줘서 MOCK 모드로 동작하고 이게 Filter가 실행되지 않게 하나보다라고 대충 넘겨짚고 webEnvironment를 지정해줘도 마찬가지..

한참을 헤메고 알게된 결론은, **MockMvc로 Filter를 테스트하려면 MockMvc 자체에 Filter를 추가해줘야 한다.**

```
@BeforeEach
public void beforeEach() {
    mvc = MockMvcBuilders.webAppContextSetup(ctx)
            .addFilters(aFilter, bFilter)
            .addFilters(cFilter)
            .alwaysDo(print())
            .build();
}
```

이렇게 오랫동안 헤멘 근본적인 이유는 MockMvc가 클라이언트만을 모사하는 거라는 잘못된 넘겨집기 때문이었다. ㅠㅜ

MockMvcTest 에서 Spring Security Filter 적용은 공식 문서 https://docs.spring.io/spring-security/site/docs/current/reference/html5/#test-mockmvc-setup 를 참고한다.
