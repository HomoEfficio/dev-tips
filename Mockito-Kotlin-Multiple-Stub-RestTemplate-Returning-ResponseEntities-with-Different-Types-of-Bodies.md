# Mockito Kotlin Multiple Stub RestTemplate Returning ResponseEntities with Different Types of Bodies

각각 다른 api 를 호출해서 다른 타입의 응답본문을 가진 ResponseEntity 를 반환하는 `Restemplate.exchange()` 메서드를 하나의 테스트에서 2회 이상 Stub 해야한다면 다음과 같은 `Any` 꼼수를 쓰면 된다.

```kotlin
val bodyWithTypeAAA: ResponseEntity<Any> =  // use Any instead of AAA
    ResponseEntity.ok()
        .body(
            AAA(
                args1,
                args2,
                ...
            )
        )

val bodyWithTypeBBB: ResponseEntity<Any> =  // use Any instead of BBB
    ResponseEntity.ok()
        .body(
            BBB(
                args1,
                args2,
                ...
            )
        )

given(
    restTemplate.exchange(
        anyMatching<URI>(),
        anyMatching<HttpMethod>(),
        anyMatching<HttpEntity<Any>>(),
        anyMatching<ParameterizedTypeReference<Any>>(),  // use Any
    )
)
    .willReturn(
        bodyWithTypeAAA,
        bodyWithTypeBBB,
    )
    

@Suppress("UNCHECKED_CAST")
fun <T> anyMatching(): T {
    Mockito.any<T>()
    return null as T
}

@Suppress("UNCHECKED_CAST")
fun <T> argMatching(matcher: ArgumentMatcher<T>): T {
    Mockito.argThat(matcher)
    return null as T
}
```

