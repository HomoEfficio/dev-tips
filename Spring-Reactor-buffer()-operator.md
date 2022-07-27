# Spring Reactor buffer() operator

`buffer(3)` 연산자를 사용하면 3개씩 요청을 하는 걸까 아니면 전부를 요청한 후 사용할 때 3개씩 잘라 사용하게 되는 걸까?

확인해보자.

```kotlin
    @Test
    fun `buffer invokes request()`() {
        Flux.range(0, 10)
//            .doOnRequest { longNum ->
//                println("onRequest $longNum")
//            }
            .buffer(3)
            .doOnRequest { longNum ->
                println("request() 호출 될 때 인자값은?: $longNum")
            }
            .log()
            .subscribe {
                println("index: $it")
            }
    }
```

실행 결과

```
2022-07-27 10:57:42.573  INFO 20899 --- [    Test worker] reactor.Flux.Peek.1                      : onSubscribe(FluxPeek.PeekSubscriber)
2022-07-27 10:57:42.576  INFO 20899 --- [    Test worker] reactor.Flux.Peek.1                      : request(unbounded)
request() 호출 될 때 인자값은?: 9223372036854775807
2022-07-27 10:57:42.588  INFO 20899 --- [    Test worker] reactor.Flux.Peek.1                      : onNext([0, 1, 2])
index: [0, 1, 2]
2022-07-27 10:57:42.589  INFO 20899 --- [    Test worker] reactor.Flux.Peek.1                      : onNext([3, 4, 5])
index: [3, 4, 5]
2022-07-27 10:57:42.589  INFO 20899 --- [    Test worker] reactor.Flux.Peek.1                      : onNext([6, 7, 8])
index: [6, 7, 8]
2022-07-27 10:57:42.589  INFO 20899 --- [    Test worker] reactor.Flux.Peek.1                      : onNext([9])
index: [9]
2022-07-27 10:57:42.590  INFO 20899 --- [    Test worker] reactor.Flux.Peek.1                      : onComplete()
```

`request(unbounded)`라고 나오고 출력해본 값도 Long.MAX_VALUE 인 9223372036854775807인 걸로 봐서는 결론은 다음과 같다.

>**`buffer(int)`를 사용해도 전체를 요청하며, 사용할 때 buffer에서 지정한 갯수만큼 잘라내서 사용한다.**
