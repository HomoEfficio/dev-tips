# Kotlin Coroutine Spring WebFluxTest

https://github.com/HomoEfficio/dev-tips/blob/master/Kotlin-Coroutine-vs-Reactive-Streams(Reactor).md 에서 Spring WebFlux 환경에서 Reactor 코드 대신 Kotlin Coroutine 코드를 사용할 수 있음을 알게 됐다.

근데 역시나 그냥 쉽게 되지는 않고 자료에서 쉽게 찾아보기 어려운 장애물을 만나게 되더라능..

먼저 WebFluxtTest를 작성하다 만나게 된 장애물은 다음과 같다.


## Stubbing suspend function

아래와 같이 무심코 테스트 코드를 작성하고 보면 컴파일 에러가 난다.

```kotlin
@WebFluxTest(AccountController::class)
class AccountControllerTest {

    @MockBean private lateinit var accountService: AccountService
    @Autowired private lateinit var wtc: WebTestClient


    @Test
    fun `account save`() {
        // 여기!! Suspend function 'save' should be called only from a coroutine or another suspend function
        `when`(accountService.save(any()))
            .thenReturn(AccountVM("1", "omwomw", 3000))

        wtc.post().uri("/accounts")
            .contentType(MediaType.APPLICATION_JSON)
            .accept(MediaType.APPLICATION_JSON)
            .body(Mono.just(AccountReq(null, "omwomw", 3000)))
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .consumeWith { println(it) }
            .jsonPath("$.accountId").isNotEmpty
            .jsonPath("$.accountName").isEqualTo("omwomw")
            .jsonPath("$.accountBalance").isEqualTo(3000)
    }
}
```

그렇다 Service 메서드가 suspend 함수이므로 코루틴이나 다른 suspend 함수 안에서만 호출 가능한데, 위 코드는 둘 다 아니기 때문에 에러다.

어쩌지?


## suspend

가장 먼저 드는 생각은.. 그냥 테스트 메서드도 suspend 로 만들어버리면 되려나? ㅋ

```kotlin
suspend fun `account save`() {

    ...
```

역시나 안 된다 ㅋ 그런데 안 되는 메시지가 좀.. 호러야 뭐야 ㅋ

![Imgur](https://i.imgur.com/Yh1gXEJ.png)


## runBlocking

그럼 runBlocking으로 감싸보면 어떨까?

```kotlin
        runBlocking {
            `when`(accountService.save(any()))
                .thenReturn(AccountVM("1", "omwomw", 3000))
        }
```

실행 결과는 다음과 같다.

```
org.mockito.exceptions.misusing.InvalidUseOfMatchersException: 
Misplaced or misused argument matcher detected here:

-> at io.homo_efficio.scratchpad.kotlin.springboot.mongo.AccountControllerTest$account save$1.invokeSuspend(AccountControllerTest.kt:25)

You cannot use argument matchers outside of verification or stubbing.
Examples of correct usage of argument matchers:
    when(mock.get(anyInt())).thenReturn(null);
    doThrow(new RuntimeException()).when(mock).someVoidMethod(anyObject());
    verify(mock).someMethod(contains("foo"))

    ...

java.lang.NullPointerException: any() must not be null
```

요는 `any()`가 null 을 반환하기 때문에 발생하는 에러다. 이건 또 어떻게 해결하나 찾아보니 https://withhamit.tistory.com/138 여기에 아주 단순한 해법이 있었다. `Mockito.any()` 대신에 다음과 같이 자체 구현한 `any()`를 사용하면 된다.

```kotlin
    private fun <T> any(): T {
        Mockito.any<T>()
        return null as T
    }
```

게다가 테스트도 통과한다!


## mockito-kotlin

그런데 이런 수작업을 늘 할 필요는 없을 것 같고 뭔가 라이브러리가 있겠지 하고 찾아보니 kotling 용으로 나온 Mockito 인 `mockito-kotlin`이 있다. 사용 방법도 아래와 같이 코틀린스러운 코드를 작성할 수 있게 해준다고 하니 맘에 든다. 다만 `mockito-kotlin`의 mock()을 사용해야 하기 때문에 Spring의 `@MockBean`과 상충되는 면이 살짝 아쉽다.

build.gradle.kts에 다음과 같이 추가해주고,

`testImplementation("org.mockito.kotlin:mockito-kotlin:VERSION")`

테스트 코드는 다음과 같이 작성할 수 있다.

```kotlin
package io.homo_efficio.scratchpad.kotlin.springboot.mongo

import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.mockito.kotlin.any
import org.mockito.kotlin.doReturn
import org.mockito.kotlin.mock
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.boot.test.mock.mockito.MockBean
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.body
import reactor.core.publisher.Mono

@WebFluxTest(AccountController::class)
internal class AccountControllerTest {

    @MockBean lateinit var accountService: AccountService
    @Autowired lateinit var wtc: WebTestClient


    @BeforeEach
    fun beforeEach() {
        accountService = mock()  // 조금 이상하지만 mockito-kotlin 의 mock()을 사용해야 아래와 같이 coroutine mocking 가능
    }

    @Test
    fun `account save`() {

        // Mock 부분 - mockito-kotlin 덕에 코틀린스럽게 mocking 코드를 작성할 수 있다.
        // mockito-kotlin 의 mock()을 사용해야 아래와 같이 coroutine mocking 가능하므로
        // `@MockBean` 변수에 재할당하는 부분이 아쉬움
        accountService = mock {
            onBlocking { save(any()) } doReturn
                    AccountVM("1", "omwomw", 3000)
        }


        wtc.post().uri("/accounts")
            .contentType(MediaType.APPLICATION_JSON)
            .accept(MediaType.APPLICATION_JSON)
            .body(Mono.just(AccountReq(null, "omwomw", 3000)))
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .consumeWith { println(it) }
            .jsonPath("$.accountId").isNotEmpty
            .jsonPath("$.accountName").isEqualTo("omwomw")
            .jsonPath("$.accountBalance").isEqualTo(3000)
    }

}

```

`any()` 관련 이슈도 없다. 그런데 아쉽게도 테스트를 돌려보면 아래와 같이 실패한다. mocking 하면서 반환하기로 정해놓은 값이 반환되지 않기 때문이다.

```

> POST /accounts
> WebTestClient-Request-Id: [1]
> Content-Type: [application/json]
> Accept: [application/json]
> Content-Length: [79]

{
  "accountId" : null,
  "accountName" : "omwomw",
  "accountBalance" : 3000
}

< 200 OK OK
< 

0 bytes of content (unknown content-type).


java.lang.AssertionError: No value at JSON path "$.accountId"
```

왜 정해진대로 반환하지 않고 아무 값도 반환하지 않은 것일까? 현재로서는 이유를 모르겠다. 나중에 알게 되면 보완하기로 하고 일단 패스.


## springmockk

더 찾아보니 `springmockk`라는 게 있다. 맨 뒤에 아마도 kotlin을 의미하는 것 같은 k가 하나 더 붙어있다.

`springmockk`는 아예 mockito를 대체하는 놈이므로 다음과 같이 starter-test에서 mockito-core를 제거해줘야 한다.

```
// build.gradle.kts
testImplementation("org.springframework.boot:spring-boot-starter-test") {
    exclude(module = "mockito-core")
}
testImplementation("org.junit.jupiter:junit-jupiter-api")
testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine")
testImplementation("com.ninja-squad:springmockk:3.0.1")
```

테스트 코드는 다음과 같이 작성할 수 있다.

```kotlin
package io.homo_efficio.scratchpad.kotlin.springboot.mongo

import com.ninjasquad.springmockk.MockkBean
import io.mockk.coEvery
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.test.web.reactive.server.body
import reactor.core.publisher.Mono

@WebFluxTest(AccountController::class)
internal class AccountControllerTest {

    @MockkBean private lateinit var accountService: AccountService
    @Autowired private lateinit var wtc: WebTestClient


    @Test
    fun `account save`() {
        // Mockito의 `when` 대신에 `coEvery` 를 사용한다. 
        coEvery { accountService.save(any()) }
            .returns(AccountVM("1", "omwomw", 3000))


        wtc.post().uri("/accounts")
            .contentType(MediaType.APPLICATION_JSON)
            .accept(MediaType.APPLICATION_JSON)
            .body(Mono.just(AccountReq(null, "omwomw", 3000)))
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .consumeWith { println(it) }
            .jsonPath("$.accountId").isNotEmpty
            .jsonPath("$.accountName").isEqualTo("omwomw")
            .jsonPath("$.accountBalance").isEqualTo(3000)
    }

}

```

이제 테스트도 통과한다.


## 마무리

앞으로 또 어떤 장애물을 만날지 모르지만, 코틀린 코루틴과 Spring WebFlux를 함께 사용할 때는 둘 중 한 가지 방식을 택하면 된다.

> `runBlocking {}` + 자체 구현 any()
>
> springmockk





