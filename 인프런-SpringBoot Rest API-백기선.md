# 백기선 SpringBoot REST API

https://www.inflearn.com/course/spring_rest-api/ 수강 중 참고할 만한 내용 요약

## HATEOAS, HAL

TODO

## 컨트롤러 테스트

### 슬라이싱 테스트

- `@WebMvcTest` + `@MockBean` + Mockito 사용
- Mock DispatcherSevlet을 통해 Request/Response 처리
- 슬라이싱 테스트라서 서비스나 리포지토리 관련 빈은 주입해주지 않음
    - 따라서 `@MockBean`과 Mockito를 써서 서비스나 리포지토리의 동작을 Stubbing 해야한다.

- 코드 쪼가리

    ```java
    // value() 안에 Matcher 사용 가능
    MockMvc.perform(...)
        .andExpect(jsonPath("id").value(Matchers.not(100)))
        ...
    ```

### 약식 통합 테스트

- `@SpringBootTest` + `@AutoConfigureMockMvc`
- Mock DispatcherServlet을 사용하지만 서비스나 리포지토리는 실제 Bean을 주입 받아 사용 가능
- 귀찮은 Mocking, Stubbing을 피할 수 있는 장점이 있지만 실제 DB에까지 저장하므로 수행 시간은 오래 걸릴 수 있음

### Validation

- JSR-303은 데이터 항목 하나에 대한 형식적 유효성(empty, max, min 등) 검증을 담당
- 여러 데이터 항목에 걸친 유효성 검증(시작 날짜와 종료 날짜 등) 등은 별도의 Custom Validator를 만들어야 함
- 저장된 데이터와 교차 검증이 필요한 경우(이메일 중복 여부 등)에는 서비스에서 검증해야 함


### 기타

#### DTO에 없는 속성이 들어올 때 Bad Request 처리

```
# application.yml

spring.jackson.deserialization.fail-on-unknown-properties: true
```

#### 테스트 설명용 Custom Annotation

- 테스트 이름에 테스트의 의도/동작 등을 모두 담기 어려우므로 커스텀 애노테이션 사용

	```java
	@Target(ElementType.METHOD)
	@Retention(RetentionPolicy.SOURCE)
	public @interface TestDescription {		
		String value();
	}
	```

- JUnit5 에는 테스트 설명 애노테이션이 제공됨



