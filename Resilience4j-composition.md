# Resilience4j-composition

Resilience4j에 나오는 여러 장애 내성 수단은 조합해서 사용할 수 있다.  

## 조합 지정 순서와 적용 순서

조합할 때는 어떤 순서에 따라 적용될까?

### 애노테이션 방식

애노테이션 방식일 때는 **애노테이션 작성 순서와 무관하게 [기본 순서](https://resilience4j.readme.io/docs/getting-started-3#aspect-order) 가 정해져있다.**

```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( Function ) ) ) ) )
```

Bulkhead가 가장 먼저 적용되고 Retry 가 가장 나중에 적용된다.

하지만 기본 순서는 고정이 아니며 다르게 지정할 수도 있다. 숫자가 큰 쪽이 먼저 적용된다.  
아래와 같이 지정하면 기본 순서와는 다르게 Retry가 CircuitBreaker 보다 먼저 적용된다.

```yml
resilience4j:
  circuitbreaker:
    circuitBreakerAspectOrder: 1
  retry:
    retryAspectOrder: 2
```

애노테이션 방식은 문서에 명확하게 니와 있으므로 따로 실험을 해보지는 않았다.


### 데코레이션 방식

데코레이션 방식에서는 어떨까? **지정한 순서와 반대로 적용된다.**

```kotlin
val decoratedSupplier = Decorators.ofSupplier(yourSupplier)
    .withRetry(retry)
    .withCircuitBreaker(circuitBreaker)
    .withBulkhead(bulkhead)
    .decorate()

val result = decoratedSupplier.get()
```

위와 같이 돼 있으면 Bulkhead -> CircuitBreaker -> Retry 순서로 적용된다.

데코레이션을 조합한다는 것은 가장 먼저 데코레이트 한 것을 다른 데코레이션으로 감싸고 또 다른 데코레이션으로 감싸는 것이므로, 위 코드는 결국 아래와 비슷하기 때문이다.

```
val decoratorSupplier = bulkhead(circuitBreaker(retry(yourSupplier)))
val result = decoratorSupplier.get()
```

즉 `get()`이 실행되면서 가장 바깥에 있는(가장 나중에 지정한) bulkhead가 가장 먼저 적용되고, 이어서 circuitBreaker, retry가 순서대로 적용된다.


## Fallback

fallback도 각 방식별로 하나의 fallback 메서드만 가능한 게 아니라 여러 fallback 메서드를 가질 수 있다.  
여러 fallback 메서드 중에서 예외 타입에 맞는 fallback 메서드가 실행된다.

그런데 동일한 예외에 대한 fallback 메서드가 여러개 있을 때는 어느 것 먼저 실행될까?  
물론 이렇게 모호하게 작성하면 안 되겠지만 이런 경우 먼저 등록한 메서드가 실행된다.


## 실험 코드

위 데코레이션 방식으로 조합할 때의 지정/적용 순서와 fallback 메서드 실행 순서를 확인해 본 코드는 다음과 같다.

```kotlin
import io.github.resilience4j.bulkhead.BulkheadConfig
import io.github.resilience4j.bulkhead.BulkheadRegistry
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry
import io.github.resilience4j.decorators.Decorators
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.Duration
import java.time.LocalDateTime
import java.util.concurrent.CompletableFuture
import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.CountDownLatch
import java.util.concurrent.TimeUnit

internal class Resilience4jTest {

    private val bulkheadRegistry = BulkheadRegistry.of(
        BulkheadConfig.custom()
            .maxConcurrentCalls(2)
            .maxWaitDuration(Duration.ofMillis(0))
            .fairCallHandlingStrategyEnabled(true)
            .writableStackTraceEnabled(true)
            .build()
    )

    private val testBulkHead = bulkheadRegistry.bulkhead("test")

    private val circuitBreakerRegistry = CircuitBreakerRegistry.of(
        CircuitBreakerConfig.custom()
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(3)
            .failureRateThreshold(10F)
            .maxWaitDurationInHalfOpenState(Duration.ofSeconds(10L))
            .permittedNumberOfCallsInHalfOpenState(10)
            .writableStackTraceEnabled(true)
            .build()
    )

    val threadCount = 5
    val countDownLatch = CountDownLatch(threadCount)

    private val testCircuitBreaker = circuitBreakerRegistry.circuitBreaker("test")

    private val testSupplier = {
        countDownLatch.countDown()
        val msg = "메롱~~~ countDown: ${countDownLatch.count}"
        Thread.sleep(1000L)
        if (1 == 1) throw RuntimeException(msg)
        "*** ${LocalDateTime.now()} [${Thread.currentThread().name}] Resilience4j Test ***"
    }

    @Test
    internal fun `나중에 지정한 컴포넌트가 먼저 적용되고, 컴포넌트에 등록된 fallback는 예외 조건이 같다면 먼저 등록된 것이 먼저 적용된다`() {
        val decoratedSupplier = Decorators.ofSupplier(testSupplier)
            .withCircuitBreaker(testCircuitBreaker)
            .withFallback { throwable ->
                "${LocalDateTime.now()} CIRCUIT BREAKER FIRST Throwable [${Thread.currentThread().name}] $throwable"
            }
            .withFallback { throwable ->
                "${LocalDateTime.now()} CIRCUIT BREAKER SECOND Throwable [${Thread.currentThread().name}] $throwable"
            }
            .withBulkhead(testBulkHead)
            .withFallback { throwable ->
                "${LocalDateTime.now()} BULKHEAD FIRST Throwable [${Thread.currentThread().name}] $throwable"
            }
            .withFallback { throwable ->
                "${LocalDateTime.now()} BULKHEAD SECOND Throwable [${Thread.currentThread().name}] $throwable"
            }
            .decorate()


        val results = CopyOnWriteArrayList<String>()

        for (i in 1..threadCount) {
            println("[${Thread.currentThread().name}] i: $i")
            CompletableFuture
                .supplyAsync(decoratedSupplier)
                .thenAcceptAsync {
                    println(it)
                    results.add(it)
                }
        }

        countDownLatch.await(3, TimeUnit.SECONDS)

        println("===== results =====")
        results.forEach { println(it) }

        assertThat(countDownLatch.count).isEqualTo(3)
        for (i in 0..2) {
            assertThat(results[i]).contains("BULKHEAD FIRST")
        }
        for (i in 3..4) {
            assertThat(results[i]).contains("CIRCUIT BREAKER FIRST")
        }
    }
}

```

실행 결과는 다음과 같다.

```
[Test worker] i: 1
[Test worker] i: 2
[Test worker] i: 3
[Test worker] i: 4
[Test worker] i: 5
2021-12-24T17:39:40.940459 BULKHEAD FIRST Throwable [ForkJoinPool.commonPool-worker-9] io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'test' is full and does not permit further calls
2021-12-24T17:39:40.940473 BULKHEAD FIRST Throwable [ForkJoinPool.commonPool-worker-23] io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'test' is full and does not permit further calls
2021-12-24T17:39:40.940484 BULKHEAD FIRST Throwable [ForkJoinPool.commonPool-worker-27] io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'test' is full and does not permit further calls
17:39:41.947 [ForkJoinPool.commonPool-worker-19] DEBUG io.github.resilience4j.circuitbreaker.internal.CircuitBreakerStateMachine - CircuitBreaker 'test' recorded an exception as failure:
java.lang.RuntimeException: 메롱~~~ countDown: 4
    ...
17:39:41.947 [ForkJoinPool.commonPool-worker-5] DEBUG io.github.resilience4j.circuitbreaker.internal.CircuitBreakerStateMachine - CircuitBreaker 'test' recorded an exception as failure:
java.lang.RuntimeException: 메롱~~~ countDown: 3
    ...
17:39:41.954 [ForkJoinPool.commonPool-worker-5] DEBUG io.github.resilience4j.circuitbreaker.internal.CircuitBreakerStateMachine - No Consumers: Event ERROR not published
17:39:41.954 [ForkJoinPool.commonPool-worker-19] DEBUG io.github.resilience4j.circuitbreaker.internal.CircuitBreakerStateMachine - No Consumers: Event ERROR not published
2021-12-24T17:39:41.955745 CIRCUIT BREAKER FIRST Throwable [ForkJoinPool.commonPool-worker-19] java.lang.RuntimeException: 메롱~~~ countDown: 4
2021-12-24T17:39:41.955718 CIRCUIT BREAKER FIRST Throwable [ForkJoinPool.commonPool-worker-5] java.lang.RuntimeException: 메롱~~~ countDown: 3
===== results =====
2021-12-24T17:39:40.940459 BULKHEAD FIRST Throwable [ForkJoinPool.commonPool-worker-9] io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'test' is full and does not permit further calls
2021-12-24T17:39:40.940473 BULKHEAD FIRST Throwable [ForkJoinPool.commonPool-worker-23] io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'test' is full and does not permit further calls
2021-12-24T17:39:40.940484 BULKHEAD FIRST Throwable [ForkJoinPool.commonPool-worker-27] io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'test' is full and does not permit further calls
2021-12-24T17:39:41.955745 CIRCUIT BREAKER FIRST Throwable [ForkJoinPool.commonPool-worker-19] java.lang.RuntimeException: 메롱~~~ countDown: 4
2021-12-24T17:39:41.955718 CIRCUIT BREAKER FIRST Throwable [ForkJoinPool.commonPool-worker-5] java.lang.RuntimeException: 메롱~~~ countDown: 3
```

