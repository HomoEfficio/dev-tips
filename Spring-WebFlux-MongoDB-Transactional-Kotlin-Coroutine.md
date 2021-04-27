# Spring WebFlux MongoDB @Transactional Kotlin coroutine

스프링 웹플럭스에서 사용하는 Reactor 환경 + MongoDB 에서 `@Transactional`로 간단하게 트랜잭션 관리를 할 수 있을까?

**결론부터 말하면 된다. 물론 아래에 설명할 몇 가지 사전 요건을 갖춰야만 된다.**

그런데 2021-04-27 현재도 이게 된다고 간명하게 표현된 자료를 좀처럼 볼 수가 없다. 아는 주변분들께 물어도 마찬가지.  
그래서 정말 되는 거 맞는지, 또 엉터리 테스트를 작성하고 된다고 믿고 있는 건 아닐까 심히 우려될 정도.. =3=3


## 사전 요건

간단하다. MongoDB 설정 시 다음과 같은 Reactive Transaction 관련 빈을 만들어주기만 하면 끝.

```kotlin
@Configuration
class MongoConfig : AbstractReactiveMongoConfiguration() {

    override fun getDatabaseName(): String {
        return "my_reactive"
    }

    @Bean
    fun transactionManager(reactiveMongoDatabaseFactory: ReactiveMongoDatabaseFactory): ReactiveMongoTransactionManager {
        return ReactiveMongoTransactionManager(reactiveMongoDatabaseFactory)
    }

    @Bean
    fun transactionalOperator(reactiveMongoTransactionManager: ReactiveMongoTransactionManager): TransactionalOperator {
        return TransactionalOperator.create(reactiveMongoTransactionManager)
    }
}
```


## coroutine suspend 함수

coroutine suspend 함수에도 `@Transactional`만 붙여서 간단하게 트랜잭션 관리 가능할까?

가능하다.

아래 깃헙에서 HelloServiceTest 를 실행해보면 확인할 수 있다.


https://github.com/HomoEfficio/scratchpad-kotlin-webflux.git


## 주의 사항

**혹시 테스트 작성 자체가 잘못된 걸 수도 있으니 주의 ㅋㅋ**

**혹시 잘못 작성한 거면 제발 지적 부탁**
