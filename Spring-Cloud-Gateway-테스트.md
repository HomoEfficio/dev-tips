# Spring Cloud Gateway 테스트

Spring Cloud Gateway는 마이크로서비스 패턴 중에서 Edge Server 패턴을 구현할 수 있게 해주는 라이브러리다.

지금까지 Edge Server로 Netflix Zuul 이 사용돼왔는데 [2018년 12월에 Maintenance 모드로 전환](https://spring.io/blog/2018/12/12/spring-cloud-greenwich-rc1-available-now)됐고 그 후속타로 Netflix가 아닌 Pivotal에서 만든 것이 Reactive 방식의 Spring Cloud Gateway다.

Edge Server의 기본적인 역할은 라우팅과 인증이다. 이를 테스트 하려면 라우팅된 후에 접근하게 될 upstream 서버를 흉내낼 수 있는 대역이 필요하고, 인증 테스트를 위해 Spring Security가 필요하다.

아쉽지만 당연하게도 **`@WebFluxTest` 테스트는 라우팅 설정이 반영되지 않은 테스트 서버 인스턴스가 구동되므로 조금 무겁더라도 `@SpringBootTest` 테스트에서 수행해야 한다.**

upstream 서버 대역으로는 [Spring MockRestServiceServer](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/client/MockRestServiceServer.html)도 있고, [Spring Cloud Contract WireMock](https://cloud.spring.io/spring-cloud-contract/reference/html/project-features.html#features-wiremock)도 있는데, 일단 WireMock으로 해본다. 이유는 일단 다음과 같은 친절한 에러 메시지가 마음에 들어서..

```
2020-06-07 16:58:53.296 ERROR 7557 --- [tp819330075-445] WireMock                                 : 
                                               Request was not matched
                                               =======================

-----------------------------------------------------------------------------------------------------------------------
| Closest stub                                             | Request                                                  |
-----------------------------------------------------------------------------------------------------------------------
                                                           |
POST                                                       | POST
/v1/sellers                                                | /v1/customers                                       <<<<< URL does not match
                                                           |
                                                           |
-----------------------------------------------------------------------------------------------------------------------
```

대략 다음과 같은 형식으로 테스트 할 수 있다.

```java
// server.port 로 지정된 포트로 Edge Server 인스턴스 실행
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class EdgeServerApplicationTests {

    private static final WireMockServer wireMockServer = new WireMockServer(WireMockSpring.options().port(8080));  // upstream 서버의 port 지정

    private WebTestClient client;

    @Autowired
    private Environment environment;

    @BeforeAll
    static void beforeAll() {
        wireMockServer.start();
    }

    @BeforeEach
    void beforeEach() {
        // client는 Edge Server에 요청을 보내도록 설정
        client = WebTestClient.bindToServer().baseUrl("http://localhost:" + environment.getProperty("server.port")).build();
    }

    @AfterEach
    void afterEach() {
        wireMockServer.resetAll();
    }

    @AfterAll
    static void afterAll() {
        wireMockServer.shutdown();
    }


    @DisplayName("인증 없이 Seller를 조회하면 401이 반환된다.") // 테스트 작성 당시(Security 추가 안 된 상태)에는 그냥 404 반환
    @Test
    void findSellerWithoutAuthenticated() {

        wireMockServer.stubFor(
                WireMock.get(WireMock.urlEqualTo("/v1/sellers/1"))
                        .willReturn(WireMock.aResponse().withStatus(HttpStatus.OK))
        );


        client.get()
                .uri("/v1/sellers/1")
                .exchange()
                .expectStatus().isUnauthorized();
    }

```

한 단계 더 나아가서 upstream 서버의 포트를 하드코딩하기 싫다면 다음과 같이 application.yml 에 지정된 값을 가져와서 사용하도록 바꿀 수도 있다.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class EdgeServerApplicationTests {

    // 8080 으로 하드코딩하지 않고 application.yml 에 지정된 값을 이용해서 static 변수 초기화 트릭 사용
    @Value("${upstream.monolith.port}")
    void setUpstreamServerPort(int port) {
        if (wireMockServer == null) {
            wireMockServer =
                    new WireMockServer(WireMockSpring.options().port(port));
            wireMockServer.start();
        }
    }

    private static WireMockServer wireMockServer;

    private WebTestClient client;

    @Autowired
    private Environment environment;


    @BeforeEach
    void beforeEach() {
        client = WebTestClient.bindToServer().baseUrl("http://localhost:" + environment.getProperty("server.port")).build();
    }

    @AfterEach
    void afterEach() {
        wireMockServer.resetAll();
    }

    @AfterAll
    static void afterAll() {
        if (wireMockServer != null) {
            wireMockServer.shutdown();
        }
    }

    ...
```
