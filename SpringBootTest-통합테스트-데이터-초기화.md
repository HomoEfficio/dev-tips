# SpringBootTest 통합테스트 데이터 초기화

@SpringBootTest 와 JUnit 을 함께 사용하면 통합테스트를 작성할 수 있다.

테스트 진행 시 테스트 데이터를 초기화(입력) 해둘 필요가 있는데, 클래스 당 1회만 수행돼야 하고, 그 외에도 몇 가지 걸림돌이 있어 정리해본다.


## 클래스 당 1회만 수행

'클래스 당 1회'라면 바로 생각나는 것이 @BeforeAll 인데, 아쉽게도 기본적으로는 static 메서드로 선언돼야 한다.  
따라서 Bean을 주입받을 수 없고XXXRepository 를 사용해서 데이터를 입력할 수 없다.

하지만 @BeforeAll 도 non static 으로 할 수 있는 방법이 JUnit5 에는 있다.

다음과 같이 @TestInstance 를 사용하면 @BeforeAll 메서드도 non static 으로 사용할 수 있고, 결국 Bean 도 사용할 수 있다.

```kotlin
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)  // 여기!!
@TestMethodOrder(MethodOrderer.MethodName::class)
internal class IntegrationTest {

    @Autowired
    private lateinit var xxxRepository: XxxRepository

    @BeforeAll
    internal fun beforeAll() {
        xxxRepository.save(xxx)
    }
        
//...
```

## LazyInitializationException

Lazy Loading이 적용돼 있을 경우 테스트 데이터를 입력하다가 LazyInitializationException이 발생할 수 있다.

보통은 @Transactional 을 적절히 붙여주면 발생하지 않지만 이 경우에는 붙여줘도 여전히 LazyInitializationException이 발생한다.

아래와 같이 PlatformTransactionManager를 사용하면 Lazy Loading이 적용돼 있더라도 문제 없이 데이터를 입력할 수 있다.

```kotlin
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)  // 여기!!
@TestMethodOrder(MethodOrderer.MethodName::class)
internal class IntegrationTest {

    @Autowired
    private lateinit var xxxRepository: XxxRepository

    @Autowired
    private lateinit var platformTransactionManager: PlatformTransactionManager  // 여기!!

    @BeforeAll
    internal fun beforeAll() {
        // 여기!!
        val txStatus: TransactionStatus = platformTransactionManager.getTransaction(TransactionDefinition.withDefaults())
        try {
            xxxRepository.save(xxx)
        } catch (e: Exception) {
            platformTransactionManager.rollback(txStatus)
            throw e
        }
        platformTransactionManager.commit(txStatus)
    }
        
//...
```


## SQL 파일 활용

테스트 데이터를 Repository 가 아니라 그냥 SQL문을 통해서 입력할 수도 있다. 다음과 같이 ResourceDatabasePopulator 를 사용하면 클래스패스에 있는 파일을 읽고 SQL을 실행할 수 있다.

```kotlin
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)  // 여기!!
@TestMethodOrder(MethodOrderer.MethodName::class)
internal class IntegrationTest {

    @Autowired
    private lateinit var xxxRepository: XxxRepository

    @Autowired
    private lateinit var platformTransactionManager: PlatformTransactionManager  // 여기!!

    @BeforeAll
    internal fun beforeAll() {
        // 여기!!
        val txStatus: TransactionStatus = platformTransactionManager.getTransaction(TransactionDefinition.withDefaults())
        try {
            xxxRepository.save(xxx)
        } catch (e: Exception) {
            platformTransactionManager.rollback(txStatus)
            throw e
        }
        platformTransactionManager.commit(txStatus)


        // 여기!!
        val populator1 = ResourceDatabasePopulator()
        populator1.addScript(ClassPathResource("sql-01.sql"))
        populator1.execute(dataSource)

        val populator2 = ResourceDatabasePopulator()
        populator2.addScript(ClassPathResource("sql-02.sql"))
        populator2.execute(dataSource)
        //...
    }
        
//...
```

