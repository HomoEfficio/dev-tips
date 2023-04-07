# Spring Reactor Fire and Forget + Retry

웹 애플리케이션에서는 처리가 오래 걸리는 작업의 경우 화면에서 오랫동안 응답 받지 못하고 기다리게 하는 것보다는,  
먼저 바로 응답을 줘서 화면에서 대기가 발생하지 않게 하고 실제 처리는 백그라운드에서 진행되도록 구현해야할 때가 있다.

일반적인 서블릿 환경에서는 스프링의 `@Async`를 사용해서 간단하게 처리할 수 있지만, 리액터에서는 이것도 까다롭더라능..

그래서 예제 코드를 남겨둔다.

```kotlin
fun processSomethingInBackground(param: AnyThing): Mono<Void> {
    return prepareSomething(param)
        .flatMap {
            returnImmediatelyAndDoSomethingInBackground(it, param)
            Mono.empty()
        }
}

private fun prepareSomething(param: AnyThing): Mono<Something> =
    // 생략
    

private fun returnImmediatelyAndDoSomethingInBackground(
    something: Something,
    param: AnyThing,
) {
    Mono.fromCallable {  // 백그라운드에서 실행될 별도의 Reactor Flow 생성
        doSomething(param)
    }.publishOn(Schedulers.boundedElastic())
        .retryExponentialBackoff(
            times = 10L,
            first = Duration.ofSeconds(10),
            max = Duration.ofHours(2),
        ) {
            log.warn("Retrying after ${it.backoff().toSeconds()}s, due to ${it.exception()}")
        }
        .flatMap { result ->
            postProcess(result)
        }            
        .doOnError {
            log.error("Do Something with param $param failed due to $it")
        }
        .subscribe { finalResult ->  // 반드시 별도로 subscribe 해야 의도대로 동작한다
            log.info("Do Something Succeeded with $finalResult")
        }
}

private fun doSomething(param: AnyThing): Mono<SomeResult> =
    // 생략

private fun postProcess(result: SomeResult): Mono<FinalResult> =
    // 생략
```

테스트 코드는 대략 다음 패턴을 따라 작성한다.

```kotlin
StepVerifier.create(
    // fire and forget 하는 publisher
    processSomethingInBackground(anyParam)
        .subscriberContext(context)
)
    .verifyComplete()

log.debug("+++++ 백그라운드 처리 완료될 때까지 대기 +++++")
Thread.sleep(2000)
log.debug("+++++ 단언 시작 +++++")

StepVerifier.create(
    // processSomethingInBackground()의 결과를 포함하는 데이터를 발행하는 publisher
)
    .expectNextMatches { finalResult ->
        // 처리 완료 여부 확인
    }
    .verifyComplete()
```
