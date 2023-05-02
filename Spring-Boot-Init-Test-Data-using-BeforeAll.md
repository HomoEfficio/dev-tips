# Spring Boot Init Test Data using BeforeAll

조회 테스트를 하려면 많은 데이터가 필요할 수 있다.  
이걸 테스트 할 때마다 소스 코드로 insert 하는 것도 일이라, 그냥 dump 떠놓은 SQL 스크립트로 테스트 데이터를 집어 넣을 수 있으면 편하다.  
그런데 그냥 `@Sql`로 하자니 이건 메서드 호출될 때마다 계속 실행이 되니까 낭비다.  
최소한 클래스 단위로 한 번만 스크립트가 실행되며 좋겠는데.. 하면 생각 나는 게 `@BeforeAll`이다.

그래서 대략 다음과 같이 하면, 테스트 클래스 당 1번만 실행되고, 테스트 클래스 안에 있는 모든 테스트 메서드는 입력된 데이터를 대상으로 조회 로직을 테스트 할 수 있다.

```kotlin
@ActiveProfiles("local")
@TestPropertySource(properties = [
    "spring.datasource.url=jdbc:p6spy:mysql://localhost:3306/aaa_test_query?createDatabaseIfNotExist=true&characterEncoding=UTF-8",
])
@DataJpaTest
@Import(value = [JpaConfig::class, XxxQuery::class, RestTemplate::class])
internal class XxxListQueryTest {

    @Autowired
    private lateinit var xxxQuery: XxxQuery

    @MockBean
    private lateinit var restTemplate: RestTemplate


    companion object {

        private val log = LoggerFactory.getLogger(XxxQuery::class.java)

        @JvmStatic
        @BeforeAll
        internal fun beforeAll(@Autowired dataSource: DataSource, @Autowired transactionManager: PlatformTransactionManager) {            
            TransactionTemplate(transactionManager).execute {
                try {
                    dataSource.connection.use { conn ->
                        ScriptUtils.executeSqlScript(conn, ClassPathResource("sql/data-mysql.sql"))
                        conn.commit()
                    }
                } catch (e: SQLException) {
                    log.error(e.message, e)
                }
            }
        }
    }

    ...
```


