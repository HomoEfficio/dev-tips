# Spring Boot Test 외부 서비스 호출 with @MockBean

`@SpringBootTest`를 사용해서 스프링 부트 애플리케이션 통합 테스트시 외부 서비스를 호출하는 부분에서 발목을 잡힐 수 있다.

통합 테스트라고는 해도 해당 애플리케이션 내에서의 통합 테스트가 목적이자 범위이므로, 호출 대상인 외부 서비스가 떠있지 않은 상태라면 테스트 실행 중 에러가 발생할 수 밖에 없다.

이를 위해 항상 외부 서비스를 함께 띄우는 것도 방법이겠지만, 번거롭고 생산성이 낮으며 테스트 용 외부 서비스를 띄우는 것 자체가 불가능할 수도 있다.

어쩌면 좋을까?

이 때 딱 좋은 것이 `@MockBean` 이다.

보통 `@MockBean`은 단위 테스트에서 어떤 Bean을 Mock 하기 위해서 사용해왔다. 그래서 통합 테스트에서는 Mock을 사용할 수 없을 줄 알았는데, **`@MockBean`은 `@SpringBootTest`를 사용하는 통합 테스트에서도 제 역할을 톡톡히 해낸다.**

그래서 다음과 같이 테스트 하면, 테스트 내용 중 외부 서비스를 호출하는 부분을 Mock해서 쉽게 처리할 수 있다. 아래는 스프링 부트 1.5.X 기준으 코드다.

```java
@ActiveProfiles("local")
@RunWith(SpringRunner.class)
@Transactional
@AutoConfigureMockMvc
@SpringBootTest
public class XXXControllerTest {
    ...
    @MockBean  // 외부 서비스 호출에 사용되는 RestTemplate Bean을 Mock
    private RestTemplate mockRT;
    ...
    @Test
    public void test_yyy() throws Exception {
        ...
        BDDMockito.given(mockRT.postForObject(...))  // 물론 postForObject 외에 다른 메서드도 가능
            .willReturn(YYY);
            
        ...
        
        Mockito.verify(mockRT).postForObject(...);  // 물론 postForObject 외에 다른 메서드도 가능
    }
}
```

