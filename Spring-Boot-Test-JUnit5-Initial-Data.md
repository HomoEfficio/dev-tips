# Spring Boot Test + JUnit5 에서 테스트 초기 데이터 로딩

테스트 클래스 당 1번만 데이터를 로딩해서 테스트 내내 사용하고 싶다면 어떻게 해야할까?  
쉽게 말하면 테스트 용 데이터를 어떻게 만들고 읽어서 사용할 수 있을까?

일단 Spring Boot Test 시 데이터를 로딩하는 방법은 [직접 코드를 작성하는 방법](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/testing.html#testcontext-executing-sql-programmatically)과 [`@Sql`을 사용하는 방법](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/testing.html#testcontext-executing-sql-declaratively) 2가지가 있다.

결론부터 말하면 **테스트 클래스 당 1번만 데이터를 로딩해서 계속 사용하려면 `@Sql`로는 안되고 직접 코드를 작성해야 한다.**

```java
@SpringBootTest
@Transactional
@TestInstance(TestInstance.Lifecycle.PER_CLASS)  //  (3)
class ProductReviewControllerTest {

    private MockMvc mvc;

    @Autowired
    private WebApplicationContext ctx;

    @Autowired
    private PasswordEncoder passwordEncoder;

    private JacksonTester<ProductReviewIn> productReviewInTester;

    @Autowired
    private CustomerRepository customerRepository;

    @Autowired
    private DataSource dataSource;


    @BeforeAll
    public void beforeAll() throws Exception {  // (2)
        System.out.println("BeforeAll");
        try (Connection conn = dataSource.getConnection()) {
            ScriptUtils.executeSqlScript(conn, new ClassPathResource("/data/insert-customer-seller-product.sql"));  // (1)
        }
    }


    @AfterAll
    public void afterAll() throws Exception {
        System.out.println("AfterAll");
        try (Connection conn = dataSource.getConnection()) {
            ScriptUtils.executeSqlScript(conn, new ClassPathResource("/data/truncate-customer-seller-product.sql"));
        }
    }
    
    ...
}
```

코드 작성 시 특이한 놈은 3가지다.



아래 내용은 될 것 같지만 안 된다.

```java
@SpringBootTest
@Transactional  // 테스트 메서드 종료시마다 롤백
class ProductReviewControllerTest {

    private MockMvc mvc;

    @Autowired
    private WebApplicationContext ctx;

    private JacksonTester<ProductReviewIn> productReviewInTester;

    // 모든 테스트 시작 전 한 번 실행되는 @BeforeAll 메서드에 @Sql 을 추가해서 초기 데이터 로딩
    @Sql({"/data/insert-customer-seller-product.sql"})
    @BeforeAll
    public static void beforeAll() {
    }

    // 모든 테스트 종료 시 한 번 실행되는 @AfterAll 메서드에 @Sql 을 추가해서 초기 데이터 제거
    @Sql({"/data/truncate-customer-seller-product.sql"})
    @AfterAll
    public static void afterAll() {
    }

    @BeforeEach
    public void beforeEach() {
        mvc = MockMvcBuilders.webAppContextSetup(ctx)
                .addFilters(new CharacterEncodingFilter("UTF-8", true))
                .alwaysDo(print())
                .build();
        JacksonTester.initFields(this, new ObjectMapper());
    }

    @ParameterizedTest(name = "상품 {0} 에 대한 고객 {1} 의 리뷰 {2} 생성")
    @MethodSource("productReviews")
    public void create(Long productId, Long customerId, String comment) throws Exception {
        postNewProductReview(productId, customerId, comment)
                .andExpect(status().isOk())
                .andExpect(jsonPath("productId").value(productId))
                .andExpect(jsonPath("productName").exists())
                .andExpect(jsonPath("customerId").value(customerId))
                .andExpect(jsonPath("customerName").exists())
                .andExpect(jsonPath("comment").value(comment))
        ;
    }

    private static Stream<Arguments> productReviews() {
        return Stream.of(
                Arguments.of(1L, 1L, "1 신박한 상품이네여~"),
                Arguments.of(2L, 1L, "1 이거 사려고 20년을 기다렸습니다."),
                Arguments.of(3L, 1L, "1 이 가격 말이 되나요?"),
                Arguments.of(1L, 2L, "2 보기만 해도 가슴이 벅차오릅니다."),
                Arguments.of(2L, 2L, "2 사진과 너무 다릅니다.")
        );
    }

    private ResultActions postNewProductReview(Long productId, Long customerId, String comment) throws Exception {
        return mvc.perform(post("/v1/product-reviews")
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON)
                .content(productReviewInTester.write(new ProductReviewIn(productId, customerId, comment)).getJson()));
    }
}

```

H2 용 truncate

```sql
SET REFERENTIAL_INTEGRITY FALSE;

truncate table seller;
truncate table customer;
truncate table product;

SET REFERENTIAL_INTEGRITY FALSE;

```
