# Kotlin Coroutine

코틀린 코루틴은 비동기를 다루는 다른 프로그래밍 방식인 reactive에 비해 문법적으로 훨씬 간편하지만,  
스코프(CoroutineScope), 컨텍스트(CoroutineContext), 디스패처(CoroutineDispatcher), Job 등 혼동스러운 개념들이 있어 살펴본다.

짧게 정리하면 다음과 같다.

>- 코루틴은 스코프(CoroutineScope) 안에서만 실행될 수 있다
>- 스코프는 코루틴이 실행될 수 있는 영억/범위다
>- 스코프는 기본적으로 `GlobalScope`를 사용할 수 있지만, 대부분 `GlobalScope`로부터 하위의 다른 스코프를 만들어서 사용한다
>- 컨텍스트(CoroutineContext)는 스코프의 한 property이며, 스코프 안에 있는 코루틴(들)이 스코프 안에서 전역적으로 사용될 수 있는 문맥(정보 및 함수 저장소)이다
>- 디스패처(CoroutineDispatcher)는 컨텍스트의 한 요소이며, 코루틴이 어느 스레드에서 실행/재개될지 지정할 수 있게 해준다

이 정도만 이해해도 다른 코루틴 자료를 보는 데 적지 않은 도움이 될 것이다.

여기서 말하는 코루틴은 모두 코틀린의 코루틴이며 일반적인 프로그래밍 용어인 코루틴과 동일하지 않다.  
일반적인 프로그래밍 용어로서의 코루틴은 스레드, light-weight 스레드와는 아무런 관계가 없다.


# Coroutine

>A coroutine is an instance of suspenable computation.  
>
>_from https://kotlinlang.org/docs/coroutines-basics.html#your-first-coroutine

코루틴은 코드 블록을 받아서 다른 코드와 동시에 실행된다는 점에서는 개념적으로 스레드와 비슷하지만,  
코루틴은 어떤 특정 스레드에 묶이지 않는다.  
다르게 얘기하면 코루틴 A는 B 스레드가 suspend 할 수 있고, C 스레드가 resume 할 수 있다.

코루틴도 OS 관점에서는 결국 어떤 OS 스레드 상에서 실행되지만,  
실행이 보류(suspend)되는 동안 자신이 실행된 스레드를 blocking 상대로 만들지 않으며,  
보류 후 재개될 때는 다른 스레드에서 실행될 수도 있다.  

코루틴은 실행 흐름이면서도 스레드보다 가볍다는 관점에서 light-weight 스레드라고 할 수 있지만 스레드와는 많이 다르다.


## Coroutine Builder

- 주어진 스코프(바로 아래 'CoroutineScope' 참고) 안에서 새 코루틴을 만드는 함수
- 따라서 이미 존재하는 스코프 안에서만 호출 가능
- 모든 코루틴 빌더 함수는 CoroutineScope의 확장 함수이며
- `scopeA.launch`는 `scopeA` 스코프 안에서 실행될 수 있는 새 코루틴을 생성
  - `scopeA`가 생략되면 `launch`가 호출되는 위치를 포함하는 최하위 스코프에서 새 코루틴을 생성
  - 아무런 스코프도 없는 곳에서 `launch`를 호출하면 `Unresolved reference: launch` 컴파일 에러 발생

### fun runBlocking

- https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
- GlobalScope에서 실행될 수 있는 코루틴 생성

### fun launch

- https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
- `Job` 반환

### fun async

- https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
- `Deferred<T>` 반환



# CoroutineScope

- https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
- 코루틴이 실행될 수 있는 영역/범위
  - 코루틴은 스코프 내에서만 실행될 수 있고, 아무런 스코프가 없는 곳에서는 실행될 수 없다
- 모든 스코프는 결국 GlobalScope를 뿌리로 해서 생겨나며 따라서 모든 스코프는 GlobalScope의 하위 스코프다
- 하위 스코프에서 생성된 코루틴이 완료되기 전에는 상위 스코프의 코루틴도 완료될 수 없다
- CoroutineContext는 CoroutineScope의 property 속성이다

## Scope Builder

- 새로운 스코프를 생성하는 suspend 함수(Scoping Function이라고 부르기도 한다)
  - 새로운 스코프를 만들지만 새로운 코루틴을 만들지는 않는다
- suspend 함수이므로 코루틴 안에서만 호출 가능

### Scope Buillder와 Coroutine Builder
- 코루틴을 만드는 코루틴 빌더 함수는 스코프 안에서만 호출 가능한데, 스코프를 만드는 스코프 빌더 함수는 코루틴 안에서만 호출 가능하다고 하니 이 부분이 순환 논리 같아 혼동스럽다
  - 일반적으로 GlobalScope의 코루틴 빌더 함수를 호출(`GlobalScope.launch(또는 .async, .runBlocking, ...)`)해서 글로벌 스코프에서 코루틴 A를 만들고,
  - 글로벌 스코프에서 만든 코루틴 A 안에서 suspend 함수인 스코프 빌더 함수를 호출(`coroutineScope { ... }`, `withContext(context) { ... }`)해서 글로벌 스코프 하위의 스코프 B를 만들고,
  - 이 B 스코프 안에서 다른 suspend 함수를 호출하는 방식으로 사용한다

### suspend fun coroutineScope

- https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
- 기존 스코프에 있는 컨텍스트를 상속받고, 기존 컨텍스트의 Job은 오버라이드하면서 새 스코프 생성

### suspend fun withContext

- https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
- 특정 컨텍스트를 가지는 새 스코프 생성



# CoroutineContext

- https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/
- CoroutineScope의 property로서 스코프와 생명주기를 함께 한다
- 스코프 안에 있는 코루틴(들)이 스코프 안에서 전역적으로 사용될 수 있는 문맥(정보 및 함수 저장소)
- 컨텍스트 내용 중 중요한 것은 `Job`과 `Dispatcher`
- indexed set 이라는, set과 map을 혼합한 자료 구조 사용
  - 컨텍스트의 요소는 모두 `Element`를 상속받은 타입인데 `Element` 타입도 `CoroutineContext`를 상속받은 타입이다
  - 따라서 하나의 디스패처가 곧 컨텍스트이기도 하며, `+` 연산자를 사용해서 `디스패처 + 잡 + 이름 + ...`와 같이 모두를 포함하는 컨텍스트를 만들 수도 있다
- 컨텍스트 내용은 불변이며, 스코프의 컨텍스트도 val로 선언돼 있으므로 교체 불가
  - `+`, `-`를 사용해서 컨텍스트 내용에 추가/삭제한 새로운 컨텍스트를 만들어서 `withContext`의 인자로 전달해줄 수는 있음
- 기본값으로 아무 정보도 포함하지 않는 `EmptyCoroutineContext`가 주로 사용 된다.



# Coroutine Dispatcher

- 코루틴이 어느 스레드에서 실행/재개될지 지정

## Dispatchers.Default

- 명시적으로 지정하지 않으면 사용되는 디스패처
- common pool of shared background threads에서 실행/재개
- CPU를 많이 소모하는 연산 집중 코루틴에 적합

## Dispatchers.IO

- shared pool of on-demand created threads에서 실행/재개
- File I/O, blocking socket I/O 처럼 블로킹 연산에 적합

## Dispatchers.Unconfined

- 코루틴 컨텍스트를 생성하는 현재 실행 중인 스레드에서 실행
- 재개될 때는 특정 스레드나 풀이 아니라 해당 suspend 함수가 사용하는 어떤 스레드에서도 재개될 수 있음
- **일반적인 코드에서는 사용하지 말아야 한다**
  - see [here](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html)
- 이 디스패처를 사용해서 생성되는 중첩 코루틴은 스택 오버플우를 피하기 위해 이벤트루프를 형성

## Dispatchers.Main

- UI 객체가 사용되는 main 스레드에서 실행/재개
- 보통 싱글 스레드 환경에서 사용



# Continuation

TODO



# Job

TODO

---
# 주요 함수 설명

## Coroutine Builder

### runBlocking

- `runBlocking`은 코루틴 빌더이며 코루틴이 아닌 `fun main()`과 코루틴 코드를 연결지어주는 역할을 한다
- `runBlocking`은 묵시적으로 `GlobalScope`에서 실행된다
- `runBlocking`은 인자로 전달된 코루틴 코드의 실행이 완료될 때까지 `runBlocking`이 실행되는 스레드를 점유하고 다른 일을 하지 못하게 한다
  - 그래서 `runBlocking`을 다른 코투틴 안에서 사용하면 안 되며 main 함수나 테스트에서만 사용해야 한다

따라서 다음과 같이 `runBlocking`을 다른 코드와 동등하게 섞어서 blocking이 발생하도록 사용하는 것은 좋지 않고,

```kotlin
fun main() {
    println("main start")

    runBlocking {
        println("runBlocking start")

        launch { 
            delay(1000L)
            println(" World")
        }

        println("Hello")
        println("runBlocking end")
    }
    
    println("after runBlocking")
    println("main end")
}
```

다음과 같이 main 함수 내용 전체를 감싸서 할당하는 방식으로 사용하는 것이 좋다.

```kotlin
fun main() = runBlocking {
    println("main and runBlocking start")

    launch { 
        delay(1000L)
        println(" World")
    }

    println("Hello")
        
    println("main and runBlocking end")
}
```

`runBlocking`을 테스트 함수에서 사용할 때도 마찬가지로 전체를 감싸야 하는데, 위와 같이 감싼 후 할당을 하면 JUnit 테스트 시 테스트 함수가 테스트 함수로 인식되지 않을 수도 있다.  
이럴 때는 아래와 같이 테스트 함수에 할당하는 방식 대신 테스트 함수 내부 전체를 감싸는 방식으로 작성하면 된다.

```kotlin
@Test
fun `runBlocking test`() {

    runBlocking {
        println("runBlocking start")

        launch { 
            delay(1000L)
            println(" World")
        }
        println("Hello")
            
        println("runBlocking end")
    }
}
```

### launch

`launch`는 코루틴 빌더로서 새 코루틴을 생성해서 다른 코드와 동시에 실행된다. `CoroutineScope`가 있는 곳에서만 호출 가능하다.  
따라서 아무런 CoroutineScope가 없는 곳에서 단순히 아래와 같이 호출하면 `Unresolved reference: launch` 컴파일 에러가 난다.

```kotlin
suspend fun doWorld() {
    println(tname("doWorld start"))
    launch {
        delay(1000L)
        println(tname("World!"))
    }
    println(tname("Hello"))
    println(tname("doWorld end"))
}
```

아래와 같이 `coroutineScope`를 통해 `CoroutineScope`가 만들어진 후에 `launch`를 호출할 수 있다.  
`coroutineScope`가 호출될 당시 존재하던 컨텍스트를 상속받되, 기존 컨텍스트의 Job은 새로 만들어 override 한다.

```kotlin
suspend fun doWorld() = coroutineScope {
    println(tname("doWorld start"))
    launch {
        delay(1000L)
        println(tname("World!"))
    }
    println(tname("Hello"))
    println(tname("doWorld end"))
}
```

`delay`는 suspending 함수로서 일정 시간 동안 코루틴의 실행을 보류/연기/유보(suspend)한다.
보류/연기/유보되는 동안 해당 코루틴 코드가 실행되던 스레드 A는 blocking 되지 않고 해방되어 다른 일을 수행할 수 있다.

`launch`는 `Job`을 반환하며, `Job`은 실행 취소(cancel)될 수 있다.


CoroutineScope는 위계 구조로 실행될 수 있으며, 하위 코루틴이 완료되기 전에는 상위 코루틴도 완료될 수 없다.

### async

