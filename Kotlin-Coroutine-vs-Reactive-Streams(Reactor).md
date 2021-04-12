# Kotlin Coroutine vs Reactive Streams(Reactor)

둘을 코드 수준에서 간명하게 비교한 자료(https://nexocode.com/blog/posts/reactive-streams-vs-coroutines/) 가 있어 아주 짧게 요약해 본다.

다음과 같이 사용자가 로그인 했을 때 환영 메시지를 반환하는 메서드가 있다고 할 때,

```kotlin
fun generateWelcome(
    usernameOrIp: String,
    userProfile: UserProfile?,
    block: Block?
): WelcomeMessage =
    when {
        block != null -> WelcomeMessage(
            "You are blocked. Reason: ${block.reason}",
            WARNING
        )
        else -> WelcomeMessage(
            "Hello ${userProfile?.fullName ?: usernameOrIp}",
            INFO
        )
    }
```

이를 호출해서 환영 메시지를 반환하는 코드를 명령형(Imperative), 리액티브 스트림, 코틀린 코루틴으로 각각 작성해서 비교해보자.


## Imperative Code

```kotlin
class WelcomeService {
    fun welcome(usernameOrIp: String): WelcomeMessage {
        val userProfile: UserProfile? = findUserProfile(usernameOrIp)
        val block: Block? = findBlock(usernameOrIp)
        return generateWelcome(usernameOrIp, userProfile, block)
    }
}
```

뭐 설명할 게 별로 없다. 사용자 프로파일을 가져오고 차단 사용자 여부를 확인해서 환영 메시지를 만드는 메서드를 호출하고 환영 메시지를 받은 후 반환한다. 비즈니스 로직 그대로다.


## Reactive Streams(Reactor) Code


```kotlin
fun welcome(usernameOrIp: String): Mono<WelcomeMessage> {
    return userProfileRepository.findById(usernameOrIp)
        .zipWith(blockRepository.findById(usernameOrIp))
        .map { tuple ->
            generateWelcome(usernameOrIp, tuple.t1, tuple.t2)
        }
}
```

리액티브 스트림이라고는 하지만 이것도 뭐 대충 알아보겠는데?

근데 이 코드는 의도대로 동작하지 않을 수 있다. `zip` 연산자는 인자가 비어있을 때 리액티브 스트림 후속 과정을 실행하지 않고 취소한다. 그래서 사용자 프로파일이나 차단 여부 정보 중 한 가지라도 없는 경우에는 `generateWelcome()`이 실행되지 않는다. 명령형 코드에서는 코틀린 컴파일러가 이런 케이스에 대해 처리하도록 알려주지만, 리액터를 사용하는 코드에서는 그런 장점이 사라지고 개발자 스스로 알아서 해당 케이스를 챙겨야 한다.

하나라도 비어 있는 케이스에 대한 처리 로직을 추가해서 바르게 동작하도록 만든 코드는 다음과 같다. 슬슬 안드로메다로 가는 것 같다.

```kotlin
fun welcome(usernameOrIp: String): Mono<WelcomeMessage> {
    return userProfileRepository.findById(usernameOrIp)
        .map { Optional.of(it) }
        .defaultIfEmpty(Optional.empty())
        .zipWith(blockRepository.findById(usernameOrIp)
            .map { Optional.of(it) }
            .defaultIfEmpty(Optional.empty()))
        .map { tuple ->
            generateWelcome(
                usernameOrIp, tuple.t1.orElse(null), tuple.t2.orElse(null)
            )
        }
}
```

이 많은 코드 중에서 실제 비즈니스 로직과 관련이 있는 것만 따로 구별해서 보면 다음 그림과 같다.

![Imgur](https://i.imgur.com/TC2mF5H.png)

진한 글자로 표시된 코드만 실제 비즈니스 로직이고 나머지 엄청난 양의 외계어는 비즈니스 로직과 사실상 무관하다. 이렇게 보니 정말 깊은 빡침이..


## Kotlin Coroutine Code

```kotlin
suspend fun welcome(usernameOrIp: String): WelcomeMessage {
    val userProfile = userProfileRepository.findById(usernameOrIp).awaitFirstOrNull()
    val block = blockRepository.findById(usernameOrIp).awaitFirstOrNull()
    return generateWelcome(usernameOrIp, userProfile, block)
}
```

코틀린 코루틴 코드는 `suspend`, `awaitFirstOrNull()`외에는 명령형 코드와 거의 같다. `userProfileRepository.findById(usernameOrIp)`와 `blockRepository.findById(usernameOrIp)`와 같이 데이터 저장소에서 데이터를 가져올 때 비동기/논블로킹 방식으로 처리해서 자원 효율성을 높이는데도 코드는 동기/블로킹 방식과 거의 차이가 없다.

이 정도면 리액티브 스트림과는 비교할 수 없을 정도로 훠얼씬 편리해 보인다.


## Kotlin Coroutine Reactor Code

`kotlinx-coroutines-reactor`를 사용하면 코틀린 코루틴과 리액터를 함께 사용할 수 있다.

```kotlin
@RestController
class WelcomeController(
    private val welcomeService: WelcomeService
) {
    @GetMapping("/welcome")
    suspend fun welcome(@RequestParam ip: String) =
        welcomeService.welcome(ip)
}
```

스프링 웹플럭스는 suspend 함수 결과를 받아서 반환하는 기능을 지원하고 있어서 웹플럭스 컨트롤러에서 suspend 함수를 사용할 수 있다.

그리고 코틀린 코루틴도 리액터 타입인 Mono를 지원해서 다음과 같이 섞어쓸 수 있다.

```kotlin

fun welcome(usernameOrIp: String): Mono<WelcomeMessage> {
    return mono {
        val userProfile = userProfileRepository.findById(usernameOrIp).awaitFirstOrNull()
        val block = blockRepository.findById(usernameOrIp).awaitFirstOrNull()
        generateWelcome(usernameOrIp, userProfile, block)
    }
}
```

코틀린 코루틴이 Mono는 지원하지만 아쉽게도 2021-04-12 현재 Flux 지원은 실험 상태다.

![Imgur](https://i.imgur.com/BcFIJz8.png)
