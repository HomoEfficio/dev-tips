# Counter-Intuitive Reactive Streams

비동기 프로그래밍은 늘 어렵다.

이미 오래된 애기지만 Reactive Streams의 구현체인 RxJava나 Reactor가 나오고 Spring에서도 WebFlux가 나오면서 저변이 더욱 확대된 것 같다.  
학습에 의해 넘을 수 있다고는 하지만 그것도 일부 잘 하는 개발자들에 대한 얘기같고, 현실적으로는 나같은 일반적인 개발자의 직관에 반하는 부분들이 많아 여전히 어렵고 고통스럽다.  
저변이 확대되는 것은 좋지만, 일부 개발자는 어쩌면 겉멋을 부리려고 굳이 쓰지 않아도 되는 곳에 사용하고, 이력서에 Reactor/Reactive 개발 경험을 추가하고 그걸 발판 삼아 회사를 떠난다.  
고통은 남아서 이어 받는 사람들의 몫..

자바라면 아직까지는 답이 없고(토비님이 알려주신 https://github.com/vsilaev/tascalate-async-await 같은 서드파티 라이브러리가 있기는 하다),  
코틀린이라면 다행히 코루틴(coroutine)이 있다.

둘을 비교해서 코루틴이 쉽고 직관적이라는 자료는 이미 많이 있지만, 그냥 하나 더 덧붙여본다.

## Reactor

```kotlin
    fun fluxRangeSample(): Mono<String> {
        val strList: MutableList<String> = mutableListOf()

        return Flux.range(0, 2)
            .doOnNext { outer ->
                println("outer loop index: $outer")
        
                Flux.range(1, 4)
                    .subscribe { inner ->
                        println("  inner loop i: $inner")
                        
                        Mono.just("    DB 호출 없는 Mono: $inner")
                            .subscribe { anyStr -> println(anyStr) }
                        
                        memberRepository.findById("moid117")
                            .subscribe { member ->
                                println("    DB 호출 있는 Mono: ${member.oid} - ${member.name}")
                                strList.add("${member.oid} - ${member.name}")
                            }
                    }
            }
            .doOnNext {
                println("Second doOnNext")
            }
            .doOnComplete {
                for (i in 1..5) {
                    println("doOnComplete i: $i")
                }
            }
            .then(Mono.just("[${strList.joinToString(" | ")}]"))
```

사실 보기만 해도 보고 싶은 마음이 별로 들지 않는다. 하지만 봐야 하니 참고 보면, 대충 2중 루프 돌면서 DB에서 읽은 문자열을 `strList`에 추가해서 반환하는 코드다.

그래서 아래와 같은 컨트롤러를 붙여서 호출하면 `|`로 구분된 문자열 목록이 나올 것이다.

```kotlin
    @GetMapping("/flux-range")
    fun fluxRange(): Mono<ResponseEntity<String>> {
        return memberCommandService.fluxRangeSample()
            .map { ResponseEntity.ok(it) }
    }
```

실행해보면 짠~ 하고

```
~ 🦑🍕🍺 ❯ http GET :8080/api/members/flux-range
HTTP/1.1 200 OK
Content-Length: 2
Content-Type: text/plain;charset=UTF-8

[]
```

예상과는 다르게 공허하게 비어있는 리스트가 반환된다. 하지만 로그에 보면 아래와 같이 DB에서 읽은 값도 출력이 된다.

```
outer loop index: 0
  inner loop i: 1
    DB 호출 없는 Mono: 1
  inner loop i: 2
    DB 호출 없는 Mono: 2
  inner loop i: 3
    DB 호출 없는 Mono: 3
  inner loop i: 4
    DB 호출 없는 Mono: 4
Second doOnNext
outer loop index: 1
  inner loop i: 1
    DB 호출 없는 Mono: 1
  inner loop i: 2
    DB 호출 없는 Mono: 2
  inner loop i: 3
    DB 호출 없는 Mono: 3
  inner loop i: 4
    DB 호출 없는 Mono: 4
Second doOnNext
doOnComplete i: 1
doOnComplete i: 2
doOnComplete i: 3
doOnComplete i: 4
doOnComplete i: 5
    DB 호출 있는 Mono: moid117 - 문어
    DB 호출 있는 Mono: moid117 - 문어
    DB 호출 있는 Mono: moid117 - 문어
    DB 호출 있는 Mono: moid117 - 문어
    DB 호출 있는 Mono: moid117 - 문어
    DB 호출 있는 Mono: moid117 - 문어
    DB 호출 있는 Mono: moid117 - 문어
```

'DB 호출 없는 Mono'는 예상대로 로그에 찍혀 있지만, 함께 찍힐 것이라고 기대했던 'DB 호출 있는 Mono'는 가장 마지막에 찍혀 있다.

코드를 보면 DB 조회를 모두 완료한 후에 이름처럼 `doOnComplete(...)`과 `then(...)`이 실행될 것 같지만, 실제로는 DB 조회 결과 후 리스트에 추가하는 부분이 가장 나중에 실행된 것이다.  
심지어 메서드가 반환된 다음에 실행되기 때문에 비어있는 리스트가 반환됐다.

그뿐만이 아니다. `Flux.range(0, 2).doOnNext(...)`는 예상대로 0, 1만 반복하지만, `Flux.range(1, 4).subscribe(...)`는 예상과 달리 1, 2, 3에서 끝나지 않고 4까지 반복된다. range()의 두 번째 인자가 inclusive 인지 exclusive 인지도 그때그때 달라요 처럼 보인다.

이처럼 직관과 다른 부분이 Reactor(RxJava도 마찬가지) 사용을 힘들게 만든다. 물론 앞서 말했 듯 학습을 통해 제대로 된 사용법을 익히면 `|`로 구분된 문자열을 반환하도록 수정할 수 있겠지만 그 학습 비용이 만만치 않다. 게다가 대안이 있다면 더더욱 그 비용은 낭비다.


## Coroutine

똑같은 일을 하는 코루틴 코드를 보자. `suspend`와 `awaitSingle()`을 제외하면 늘 봐오던 코드와 다를 게 없이 편안하다.

```kotlin
    suspend fun asyncAwaitRange(): String {
        val strList: MutableList<String> = mutableListOf()

        for (outer in 0..1) {
            println("outer loop index: $outer")

            for (inner in 1..4) {
                println("  inner loop i: $inner")
                println("    DB 호출 없는 Mono: $inner")
                val member = memberRepository.findById("moid117").awaitSingle()
                println("    DB 호출 있는 Mono: ${member.oid} - ${member.name}")
                strList.add("${member.oid} - ${member.name}")
            }

            println("Second doOnNext")
        }

        for (i in 1..5) {
            println("doOnComplete i: $i")
        }

        return "[${strList.joinToString(" | ")}]"
    }
```

아래와 같은 컨트롤러를 붙여서,

```kotlin
    @GetMapping("/async-await-range")
    suspend fun asyncAwaitRange(): ResponseEntity<String> {
        return ResponseEntity.ok(memberCommandService.asyncAwaitRange())
    }
```

아래와 같이 호출해보면 의도했던 것처럼 문자열 목록이 나온다.

```
~ 🦑🍕🍺 ❯ http GET :8080/api/members/async-await-range
HTTP/1.1 200 OK
Content-Length: 151
Content-Type: text/plain;charset=UTF-8

[moid117 - 문어 | moid117 - 문어 | moid117 - 문어 | moid117 - 문어 | moid117 - 문어 | moid117 - 문어 | moid117 - 문어 | moid117 - 문어]
```

로그도 직관에서 한 치의 벗어남이 없다.

```
outer loop index: 0
  inner loop i: 1
    DB 호출 없는 Mono: 1
    DB 호출 있는 Mono: moid117 - 문어
  inner loop i: 2
    DB 호출 없는 Mono: 2
    DB 호출 있는 Mono: moid117 - 문어
  inner loop i: 3
    DB 호출 없는 Mono: 3
    DB 호출 있는 Mono: moid117 - 문어
  inner loop i: 4
    DB 호출 없는 Mono: 4
    DB 호출 있는 Mono: moid117 - 문어
Second doOnNext
outer loop index: 1
  inner loop i: 1
    DB 호출 없는 Mono: 1
    DB 호출 있는 Mono: moid117 - 문어
  inner loop i: 2
    DB 호출 없는 Mono: 2
    DB 호출 있는 Mono: moid117 - 문어
  inner loop i: 3
    DB 호출 없는 Mono: 3
    DB 호출 있는 Mono: moid117 - 문어
  inner loop i: 4
    DB 호출 없는 Mono: 4
    DB 호출 있는 Mono: moid117 - 문어
Second doOnNext
doOnComplete i: 1
doOnComplete i: 2
doOnComplete i: 3
doOnComplete i: 4
doOnComplete i: 5
```


## 마무리

직관에서 많이 벗어나는 Reactive Streams는 써야만 할 때만 쓰자.

써야만 할 때가 언제일까?

단순하게 얘기하면

>**선형적인 처리 흐름인데 그 흐름에 들어있는 단위 과정 중간중간에 자원 낭비를 유발하는 blocking 구간이 있어서 이로 인한 낭비를 줄여야 하고 배압(back pressure) 관리가 필요할 때**

라고 할 수 있을 것 같다.

물론 선형적인 흐름이 아니라도 `Mono.zip()` 등 여러가지 (너무) 다양한 리액티브 연산자를 동원하면서 의도대로 실행되게 작성할 수는 있겠지만, 할 수 있다고 해서 무조건 하는 거랑 해야만 할 때 하는 거랑은 큰 차이가 있다.

이 글에서는 언급하지 않았지만 Reactive Streams에서 강조하는 또 한 가지 요소는 배압(back pressure)다. 일반적인 웹 서비스 클라이언트는 배압 관리을 필요로 하지 않는다.

그래서 위 사례처럼 선형적인 처리가 아니라 여러 중첩된 처리 과정이 난무할 때, 그리고 배압 관리가 필요 없을 때, 그러니까 사실 상 대부분의 일반적인 서비스의 경우에는 코루틴을 사용하는 것이 편리하다고 본다. 

그리고 머지 않아(한 2년 이내?) Virtual Thread이 나온다는 것을 감안하면 RxJava나 Reactor는 많은 경우 부담스러운 레거시로 남을 가능성이 크다.
