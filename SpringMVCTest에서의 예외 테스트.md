# SpringMVCTest에서의 예외 테스트

JUnit에서 예외 발생은 보통 다음의 두 가지 방식으로 테스트 해 볼 수 있다.

>`@Test(expected = 어쩌구Exception.class)`
>
>`@Rule`로 지정한 `ExpectedException`을 활용하는 방법 - [여기 참고](https://www.javacodegeeks.com/2014/03/junit-expectedexception-rule-beyond-basics.html)

그런데 `MockMvc`를 이용해서 `andExpect()`로 테스트하는 컨트롤러 단의 SpringMVC Test에서는 위의 방법을 쓸 수 없다. **실제로 예외가 발생하더라도 예외가 밖으로 던져지지 않고 `MvcResult`에 담긴 채로 테스트가 종료되기 때문**이다.

무슨 얘기인지 코드로 살펴보자.

## BindException이 발생하는 상황

### DTO

먼저 DTO 객체 코드다.

```java
public class MemberDto {

    private Long id;

    @NotNull
    @Size(min = 8)
    private String userName;

    @NotNull
    private String email;
    
    // 이하 getter/setter
}
```

`userName`은 최소 8글자라는 제약 사항을 걸어두었다.

### Controller

다음은 MemberDto를 저장하는 Controller 코드다. `@Valid`가 붙어있어서 8글자 보다 짧은 `userName`을 가진 memberDto 객체를 저장하려면 `BindException`을 던진다. 

```java
@RequestMapping(method = {RequestMethod.POST, RequestMethod.PUT})
public ResponseEntity<MemberDto> save(@RequestBody @Valid MemberDto memberDto,
                                      BindingResult bindingResult) throws BindException {

    if (bindingResult.hasErrors()) throw new BindException(bindingResult);

    return ResponseEntity.ok(memberService.save(memberDto));
}
```

### Controller Test

다음은 Controller 테스트 코드다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MemberIntegrationTest {

    private MockMvc mockMvc;

    @Autowired
    private WebApplicationContext ctx;

    @Autowired
    private ObjectMapper objectMapper;

    @Rule
    public ExpectedException thrown = ExpectedException.none();

    @Before
    public void setup() {
        mockMvc = MockMvcBuilders.webAppContextSetup(ctx)
                .alwaysDo(print())
                .build();
    }
    
    // @Test의 expected 속성을 이용 
    @Test(expected = BindException.class)
    public void 회원등록_userName_길이부족1() throws Exception {

        MemberDto memberDto = getShortUserNameMemberDto();

        String memberDtoJson = objectMapper.writeValueAsString(memberDto);

        MvcResult result = mockMvc
                .perform(
                        post("/api/v1/members")
                                .contentType(MediaType.APPLICATION_JSON_UTF8)
                                .content(memberDtoJson))
                .andReturn();
    }

    // ExpectedException을 이용하는 방법
    @Test
    public void 회원등록_userName_길이부족2() throws Exception {

        // ExpectedException으로 예외 검사
        thrown.expect(BindException.class);

        MemberDto memberDto = getShortUserNameMemberDto();

        String memberDtoJson = objectMapper.writeValueAsString(memberDto);

        MvcResult result = mockMvc
                .perform(
                        post("/api/v1/members")
                                .contentType(MediaType.APPLICATION_JSON_UTF8)
                                .content(memberDtoJson))
                .andReturn();
    }

    private MemberDto getShortUserNameMemberDto() {
        String userName = "1234567";
        String email = "homo.efficio@gmail.com";
        MemberDto memberDto = new MemberDto();
        memberDto.setUserName(userName);
        memberDto.setEmail(email);
        return memberDto;
    }
```

실행해보면 로그에 아래와 같은 내용이 찍히는 걸로 보아 Controller가 `BindException`를 던지는 것은 확실하다.

```
Resolved Exception:
             Type = org.springframework.validation.BindException
```

이처럼 Controller에서는 `BindException`을 던지는데도 불구하고, 예외가 발생하면 통과해야 할 두 개의 테스트 코드는 모두 실패한다.

이유는 앞에서 말한대로 **실제로 예외가 발생하더라도 예외가 밖으로 던져지지 않고 `MvcResult`에 담긴 채로 테스트가 종료되기 때문**이다.

그럼 어떻게 해야 예외 발생을 테스트 할 수 있을까?

## andExpect() 안에서 예외 검사

`MockMvc.perform()`를 통해서 HTTP Request를 테스트하려면 `ResultActions`에 있는 `andExpect()`, `andReturn()`, `andDo()` 이 세 가지 메서드를 사용하는 수 밖에는 없다. 예상값과 실제값을 비교 검사하는 로직은 그 중에서도 `andExpect()`를 사용해야 한다.

**Controller에서 던져진 예외는 `MvcResult.getResolvedException()`으로 알아낼 수 있다**. 그리고 `andExpect()`는 `ResultMatch`를 인자로 받는데, **`ResultMatch`는 `@FunctionalInterface`로 명시되어 있지는 않지만 구현되지 않은 메서드가 단 하나 있는 인터페이스**다. 따라서 **테스트 로직을 람다를 사용해서 `andExpect()`에 전달**할 수 있다.

```java
@Test
public void 회원등록_userName_길이부족() throws Exception {

    MemberDto memberDto = getShortUserNameMemberDto();

    String memberDtoJson = objectMapper.writeValueAsString(memberDto);

    MvcResult result = mockMvc
            .perform(
                    post("/api/v1/members")
                            .contentType(MediaType.APPLICATION_JSON_UTF8)
                            .content(memberDtoJson))
            .andExpect(
                    // assert로 예외를 검사하는 람다 사용 
                    (rslt) -> assertTrue(rslt.getResolvedException().getClass().isAssignableFrom(BindException.class))
            )
            .andReturn();
}
```

위 코드에서는 예외 클래스의 비교에 `isAssignableFrom()`을 사용했는데, 다음과 같이 CanonicalName으로 비교하는 방법도 가능하다.

```java
(rslt) -> assertEquals(
        rslt.getResolvedException().getClass().getCanonicalName(),
        BindException.class.getCanonicalName()
)
```

## 정리

>Controller 테스트에서는 던져진 예외가 `MvcResult` 내에 담겨지기 때문에 `@Test`나 `ExpectedException`으로 예외 발생을 테스트 할 수 없다.
>
>- 던져진 예외는 `MvcResult.getResolvedException()`으로 가져올 수 있고,
>- `ResultActions.andExpect()`는 추상 메서드가 하나인 인터페이스인 `ResultMatch`를 인자로 받으므로,
>- `MvcResult.getResolvedException()`으로 가져온 예외를 비교 검사하는 로직을 람다로 구현해서 `ResultActions.andExpect()`에 전달하면,
>- Controller 단에서의 예외 발생도 테스트 할 수 있다.

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
