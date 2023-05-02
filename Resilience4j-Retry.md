# Resilience4j-Retry

## retry

retry 개념은 딱히 설명이 필요없을 정도로 간단하다. 에러가 발생하면 바로 에러로 종결짓지 않고 몇 번 더 시도해보는 것을 말한다.

## 주의!!

개념적으로는 쉽지만 주의해야할 굉장히 중요한 포인트가 2개 있다.

1. **블로킹 방식의 api에 retry를 적용하면 retry가 실행되는 시간 동안 해당 스레드가 계속 점유된다.**

    예를 들어 '1초마다 최대 5번 재시도'하도록 지정하고 retry를 적용하면 첫 호출 후 4초 동안 해당 스레드가 블로킹 상태로 점유되어 자원 효율성이 떨어질 수 있다는 점을 명심해야 한다.

2. 여러번 실행되더라도 비즈니스 처리에 악영향을 미치지 않는 **멱등(idempotent)한 api 호출에만 적용해야 한다.**

    예를 들어 외부 서비스의 상태 변경은 정상 처리됐지만 응답을 받는 과정에서 네트워크 에러가 발생한다면, 클라이언트는 에러로 인지하고 재시도해서 외부 서비스를 다시 호출하게 된다. 멱등하다면 retry를 통해 계속 호출돼도 관계없지만 멱등하지 않다면 의도하지 않은 결과가 발생한다.

```kotlin
    private val cacheUpdateRetry: Retry = RetryRegistry.of(
        RetryConfig.custom<RetryConfig>()
            .retryExceptions(Throwable::class.java)
            .maxAttempts(5)
            .intervalFunction(
                IntervalFunction.ofExponentialBackoff(
                    Duration.ofMillis(200),  // 처음에 200ms 후에 retry
                    2.0,  // 실패 시 마다 재호출 간격 * 2
                    Duration.ofSeconds(2)  // 최대 재호출 간격 2s
                )
            )
            .failAfterMaxAttempts(true)
            .build()
    ).retry("cacheUpdate")


    private val bulkheadRegistry = BulkheadRegistry.of(
        BulkheadConfig.custom()
            .maxConcurrentCalls(30)
            .maxWaitDuration(Duration.ofMillis(500L))
            .fairCallHandlingStrategyEnabled(true)
            .writableStackTraceEnabled(true)
            .build()
    )

    private val cacheUpdateBulkhead =
        bulkheadRegistry.bulkhead("cacheUpdate")


    fun updateCache(
        restTemplate: RestTemplate,
        url: String,
        authKey: String,
    ) {
        val cacheUpdateRunnable: () -> Unit = {
            // restTemplate 등을 이용한 target api 호출
            
        }

        return Decorators.ofRunnable(cacheUpdateRunnable)
            .withRetry(cacheUpdateRetry)  // 대상 api가 멱등인 것 확인 완료 후 retry 적용
            .withBulkhead(cacheUpdateBulkhead)
            .decorate()
            .run()
            
        // 값을 받아오는 경우라면
        // otherInfoSupplier: () -> OtherInfo = {
        //   // restTemplate 등을 이용한 target api 호출 후 Response Body 반환
        // }
        // return Decorators.ofSupplier(otherInfoSupplier)
        //     .withRetry(otherRetry)  // 대상 api가 멱등인 것 확인 완료 후 retry 적용
        //     .withBulkhead(otherBulkhead)
        //     .decorate()
        //     .get()
    }
```


## non-blocking retry

TODO



