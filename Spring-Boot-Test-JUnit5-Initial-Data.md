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

    ...
    
    @Autowired
    private DataSource dataSource;


    @BeforeAll
    public void beforeAll() throws Exception {  // (2)
        System.out.println("BeforeAll");
        try (Connection conn = dataSource.getConnection()) {  // (2)
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

코드에서 특이한 놈은 3가지다.

(1)은 실제 SQL 스크립트 파일을 실행해주는 유틸인 것 외에는 크게 특이하지는 않다.  

재미있는 건 두 군데에 표시한 (2)다.  
일반적으로 `@BeforeAll`은 static 메서드로 사용된다. 그래서 `@Autowired`로 주입받은 변수를 참조해서 사용할 수 없는데, 여기에서는 `@BeforeAll`을 붙인 메서드가 static 도 아니고 그래서 주입받은 `dataSource`를 이용해서 DB connection 을 얻을 수 있다.  

어떻게 이게 가능했을까?  
해답은 (3)으로 표시한 `@TestInstance(TestInstance.Lifecycle.PER_CLASS)`에 있다.

[Baeldung 문서](https://www.baeldung.com/junit-testinstance-annotation)에 정말 잘 설명돼있는데,  
짧게 얘기하면,
- 테스트 클래스의 인스턴스는 원래는 테스트 메서드가 수행될 때마다 계속 새로 생성되지만,
- **`@TestInstance(TestInstance.Lifecycle.PER_CLASS)`를 붙여주면 새로 생성하지 않고 하나만 생성해서 테스트 클래스에 포함된 테스트 메서드가 수행되는 동안 계속 사용할 수 있다.**

테스트는 모두 독립적이어야 하므로 이렇게 테스트 클래스 인스턴스 하나를 계속 유지해서 테스트 메서드 사이에 상태를 공유하는 건 원칙적으로는 안티패턴일 수도 있는데,  
인스턴스 하나만 사용하는 것이 여러 번 생성해서 사용하는 것보다 훨씬 경제적인 것만은 확실히다. 그 경제성은 테스트 메서드의 숫자에 비례한다. 그러니 **도를 넘지 않는 수준으로만 상태를 공유 한다면 `@TestInstance(TestInstance.Lifecycle.PER_CLASS)`를 사용해서 경제성을 취하는 것도 괜찮다고 본다.**

그럼 도를 넘지 않는 수준의 경제적인 상태 공유란 뭘까?

- 테스트 데이터 구성
- 대용량 파일 로딩
- 기타 자원 로딩

위와 같이 **어떤 한 인스턴스 변수를 테스트 사이에 공유하는 게 아니라 비용이 많이 드는 자원을 로딩해서 읽기로만 사용하는 수준이라면 도를 넘지 않는 수준의 경제적인 상태 공유**라고 할 수 있다고 본다.

그래도 쓰기 가능한 상태로 공유된다면 읽기만 하겠나.. 결국에는 오염될 것이다. 그러니 이럴 때는 읽기만 수행하는 테스트 메서드만을 하나의 클래스에 따로 모아서 `@TestInstance(TestInstance.Lifecycle.PER_CLASS)`를 붙여 사용하면 된다.

---

참고로 아래와 같이 `@Sql`을 사용해도 될 것 같지만 지정한 SQL 스크립트는 실행되지 않는다.

```java
@SpringBootTest
@Transactional
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class ProductReviewControllerTest {

    ...

    @Sql({"/data/insert-customer-seller-product.sql"})
    @BeforeAll
    public void beforeAll() throws Exception {
        System.out.println("BeforeAll");
    }

    @Sql({"/data/truncate-customer-seller-product.sql"})
    @AfterAll
    public void afterAll() throws Exception {
        System.out.println("AfterAll");
    }
```



아래는 참고 - H2 용 truncate

```sql
SET REFERENTIAL_INTEGRITY FALSE;

truncate table seller;
truncate table customer;
truncate table product;

SET REFERENTIAL_INTEGRITY FALSE;

```
