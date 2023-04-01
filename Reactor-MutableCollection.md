# Reactor와 MutableCollection

다음 함수를 보자.

```kotlin
fun changeMutableInReactorFlow(): Mono<List<Item>> {
    val allowedItems = mutableListOf<Item>()
    return Flux.fromIterable(Item.values().asIterable())
        .flatMap { item ->
            restrictionPolicyRepository.findFirstByItem(item)  // 1. 얘가 항상 empty Mono를 반환한다면
                .switchIfEmpty {
                    allowedItems.add(item)  // 2. 얘가 항상 실행되면서
                    Mono.empty()
                }
        }
        .then(Mono.just(allowedItems))  // 3. allowedItems에는 항상 Item의 모든 values가 포함돼 있을 것이다.
}

enum class Item {
    SHAMPOO, TOOTH_PASTE, LOTION
}
```

주석에 써 있는 것처럼 1, 2, 3에 따라 Item enum에 있는 모든 항목이 포함된 allowedItems를 Mono에 담아 반환할 것이다.

실제로도 그렇다.


## 버그가 있다 없으니까

그런데 항상 그렇지는 않다. ㄷㄷㄷ

우리가 가장 두려워하는 그 이름 **간헐적 오류**

>![Imgur](https://i.imgur.com/4TtCgaJ.jpg)
>
>니가 있다 없으니까 숨을 쉴 수 없어  
>곁에 없으니까 머물 수도 없어  
>나는 죽어가는데 너는 지금 없는데 없는데 없는데
>
>니가 있다 없으니까 웃을 수가 없어  
>곁에 없으니까 망가져만 가는 내 모습이  
>너무 싫어 난 난 이제 기댈 곳 조차 없어

다음과 같이 1000회 반복 테스트를 해보면 10번 정도 테스트가 실패한다.

```kotlin
@RepeatedTest(value = 1000, name = "Reactor Mutable 테스트: {currentRepetition}/{totalRepetitions}")
fun testChangeMutableInReactorFlow(testInfo: TestInfo) {

    StepVerifier.create(
        changeMutableInReactorFlow()
    ).expectNextMatches {
        it.size == Item.values().size
    }
    .verifyComplete()
}
```


## 나한테 왜 이래?

![Imgur](https://i.imgur.com/B2kImVg.png)

이유는 **코틀린의 MutableCollection는 Thread-Safe 하지 않고, Reactor의 Flux.flatMap은 비동기로 여러 스레드에서 병렬 실행되기 때문**이다.

여러 가지 방식으로 해결할 수 있겠지만 두 가지만 살펴보자.


## Thread-Safe Collection 사용

가장 단순한 해법이다. 1000회 수행 시 17초 정도 걸린다.

```kotlin
fun changeMutableInReactorFlow(): Mono<List<Item>> {
    val allowedItems = Collections.synchronizedList(mutableListOf<Item>())  //// 여기!!
    return Flux.fromIterable(Item.values().asIterable())
        .flatMap { item ->
            restrictionPolicyRepository.findFirstByItem(item)
                .switchIfEmpty {
                    allowedItems.add(item)
                    Mono.empty()
                }
        }
        .then(Mono.just(allowedItems))
}
```

## Functional Style

Thread-Safe 하지 않는 MutableCollection을 아예 사용하지 않고 함수형 스타일로 새로 작성하면 다음과 같다. 16초 정도 걸린다.

```kotlin
fun changeMutableInReactorFlow(): Mono<List<Item>> {
    return Flux.fromIterable(Item.values().asIterable())
        .flatMap { item ->
            restrictionPolicyRepository.findFirstByItem(item)
                .hasElement()  // 여기부터!!
                .flatMap {
                    if (it == false) {
                        Mono.just(item)
                    } else {
                        Mono.empty()
                    }
                }
        }
        .collectList()
}
```

flatMap 안에 있는 `if-else` 식이 보기 싫으면 다음과 같이 더 깔끔해 보이게 작성할 수 있다. 다만 수행 시간은 좀 길어져서 20초 정도 걸린다.

```kotlin
fun changeMutableInReactorFlow(): Mono<List<Item>> {
    return Flux.fromIterable(Item.values().asIterable())
        .filterWhen { item ->  // 여기!!
            restrictionPolicyRepository.findFirstByItem(item)
                .hasElement()
                .map { !it }
        }
        .collectList()
}
```

## Coroutine을 쓴다면?

https://kotlinlang.org/docs/shared-mutable-state-and-concurrency.html 여기에 더 다양한 해법이 있다.
