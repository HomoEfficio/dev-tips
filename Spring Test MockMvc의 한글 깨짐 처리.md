# Spring Test MockMvc의 한글 깨짐 처리

스프링에서 테스트 코드를 작성할 때 `MockMvc`를 흔히 사용한다.

대략 아래와 같이 설정하고 사용한다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApiControllerTest {

    private MockMvc mockMvc;

    @Autowired
    private WebApplicationContext ctx;

    @Before
    public void setup() {
        this.mockMvc = MockMvcBuilders.webAppContextSetup(ctx)
                .alwaysDo(print())
                .build();
    }

    @Test
    public void 상품검색() throws Exception {
        String keyword = "sports";

        MvcResult result = this.mockMvc
                .perform(
                        get("/api/search/" + keyword)
                )
                .andExpect(status().isOk())
                .andExpect(어쩌구 Matcher...)
                .andReturn();
    }
}
```

위의 테스트 코드에서는 한글이 없으므로 아무 문제가 없는데, 아래와 같이 한글을 사용하면 깨진 한글이 Controller에 유입될 수 있으며, 결국 원하는 대로 동작하지 않게 된다.

```java
    @Test
    public void 상품검색() throws Exception {
        String keyword = "스포츠";  // 한글 사용

        MvcResult result = this.mockMvc
                .perform(get("/api/search/" + keyword))
                .andExpect(status().isOk())
                .andExpect(어쩌구 Matcher...)
                .andReturn();
    }
```

이 문제는 `MockMvc`를 설정할 때 `CharacterEncodingFilter`를 추가해주면 쉽게 해결할 수 있다.

```java
    @Before
    public void setup() {
        this.mockMvc = MockMvcBuilders.webAppContextSetup(ctx)
                .addFilters(new CharacterEncodingFilter("UTF-8", true))  // 필터 추가
                .alwaysDo(print())
                .build();
    }
```

위 방법은 `WebApplicationContext`를 주입 받아야하므로 `@SpringBootTest`에서만 사용가능하다.

`@WebMvcTest`에서는 다음과 같이 지정할 수 있다.

```kotlin
@WebMvcTest(WorldController::class)
@Import(HttpEncodingAutoConfiguration::class)
internal class WorldControllerTest {

    @Autowired lateinit var mvc: MockMvc
    
    //...
}
```

```yaml
# test/resources/application.yml
server:
  servlet:
    encoding:
      force-response: true
```

test의 application.yml 에서 저렇게 response만 따로 true로 설정해줘야 하는 이유는 Response 에 대해서는 기본값이 false 이기 때문이다.

![Imgur](https://i.imgur.com/qg3yHEA.png)
