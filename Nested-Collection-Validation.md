# Nested Collection Validation 

객체의 각 필드가 규격을 준수하는지 `@Valid`를 사용해서 간편하게 valiation 을 할 수 있다.

그런데 필드가 다른 객체를 원소로 하는 중첩 컬렉션일 때는 어떻게 validation 을 할 수 있을까?

다음과 같이 하면 중첩 컬렉션에 있는 객체의 필드도 validation 대상이 된다.

```kotlin
@PutMapping("/abc")
fun updateXxx(@RequestBody @Valid xxx: Xxx) {
    // ...
}

data class Xxx(
    // ...

    @field: [
        Min(1)
        Max(10)
    ]
    val quantity

    @Size(min = 1, max = 5)  // List의 크기 제약
    val elements: List<@Valid Yyy>,  // List 내부 원소로 사용되는 객체 validation
)


data class Yyy(
    // ...

    @NotBlank
    val name: String,
)
```


