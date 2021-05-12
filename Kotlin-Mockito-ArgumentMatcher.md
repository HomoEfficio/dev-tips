# Kotlin에서 Mockito 사용 시 주의 사항

## ArgumentMatcher

아래와 같이 Stubbing 했다. 코틀린이라서 람다식이 조금 다르고 `new`를 사용하지 않는다는 점이 다르지만 언어의 차이에서 오는 표현 방식 외에는 자바에서는 잘 돌던 코드다.

```kotlin
// findAll의 시그니처
// fun findAll(pageable: Pageable): Page<WorldOut>

given(worldQuery.findAll(argThat { it.pageNumber == 0 && it.pageSize == 2 }))
            .willReturn(PageImpl(contents, PageRequest.of(page, size), total))
```

근데 이렇게 작성하고 테스트를 실행하면 아래와 같이 친절한 에러 메시지가 나온다. 친절은 한데 너무 말이 많아서 엉뚱한 소리도 섞여있고 알아듣기는 어려운..

```
org.mockito.exceptions.misusing.InvalidUseOfMatchersException: 
Misplaced or misused argument matcher detected here:

-> 어쩌구 저쩌구 에러 발생 소스 코드 위치 알려줌

You cannot use argument matchers outside of verification or stubbing.  // 여기!! (1)
Examples of correct usage of argument matchers:
    when(mock.get(anyInt())).thenReturn(null);
    doThrow(new RuntimeException()).when(mock).someVoidMethod(anyObject());
    verify(mock).someMethod(contains("foo"))

This message may appear after an NullPointerException if the last matcher is returning an object   // 여기!! (2)
like any() but the stubbed method signature expect a primitive argument, in this case,
use primitive alternatives.
    when(mock.get(any())); // bad use, will raise NPE
    when(mock.get(anyInt())); // correct usage use

Also, this error might show up because you use argument matchers with methods that cannot be mocked.
Following methods *cannot* be stubbed/verified: final/private/equals()/hashCode().  // 이건 걍 해당 없음
Mocking methods declared on non-public parent classes is not supported.

    at org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener.resetMocks(ResetMocksTestExecutionListener.java:83) ~[spring-boot-test-2.4.5.jar:2.4.5]
    

    ... 이하 생략
```

(1)에 보면 `argument matcher`를 바르게 사용하는 용법을 친절하게 알려주는데, 용법에는 문제가 없다.

(2)에 보면 실제 stub 대상 메서드는 기본형(primitive) 인자를 받는데, 기본형이 아니라 객체를 반환하는 matcher인 `any()` 같은 걸 사용하고 그 matcher가 null을 반환하는 바람에 NullPointerException이 발생하면 지금 같은 에러가 뜰 수 있다(졸라 어렵게 얘기하네..)고 한다. 오 이게 단서가 될 것 같다.

그래서 `org.mockito.ArgumentMatchers.argThat()`가 어떻게 돼있나 보니 헐 무조건 null을 반환하게 구현돼있네??

```java
public static <T> T argThat(ArgumentMatcher<T> matcher) {
    reportMatcher(matcher);
    return null;
}
```

아 그럼 다른 matcher를 써볼까? 구체성은 확 떨어지지만 만능 치트키 같은 `org.mockito.ArgumentMatchers.any()`는 어떻게 생겼나 볼까?

```java
public static <T> T any(Class<T> type) {
    reportMatcher(new InstanceOf.VarArgAware(type, "<any " + type.getCanonicalName() + ">"));
    return defaultValue(type);
}
```

오 일단 null은 아닌데 `defaultValue(type)`을 따라가보면 `type`이 기본형이 아닐 때는 결국 null을 반환한다.

뭐여 이거.. 결국 `org.mockito.ArgumentMatchers`에 있는 애들은 코틀린에서는 못 쓰는 거네..

근데 왜 자바에서는 되고, 코틀린에서는 안 되는 거지?


## Null Safety

자바와 코틀린의 두드러진 차이 중 하나가 Null 처리니까 이쪽으로 생각해보니 답이 나왔다.

stub 대상 메서드의 파라미터는 아래와 같이 `?`이 없는 그러니까 Non Null인 `Pageable` 타입을 받게 Non Null 을 받게 돼 있다.

```kotlin
fun findAll(pageable: Pageable): Page<WorldOut>
```

그런데 `argThat()`은 null을 반환하니까 코틀린에서는 안 되고, 참조형 타입에 null을 전달해도 탈이 없는 자바에서는 됐었던 거였다.

그래서 해결은?


## 해결(이라기보다는 파묻기)

일단 파라미터를 Nullable 하게 바꾸고 테스트 코드도 그에 맞게 바꿔서 해보자.

```kotlin
fun findAll(pageable: Pageable?): Page<WorldOut> {
    return worldRepository.findAll(pageable!!).map { WorldOut.fromEntity(it) }
}

given(worldQuery.findAll(argThat { it!!.pageNumber == 0 && it.pageSize == 2 }))
    .willReturn(PageImpl(contents, PageRequest.of(page, size), total))
```

`!!`를 여기저기 넣어야 했지만, 테스트는 에러 없이 실행된다!!

근데 이대로 좋은가? 겨우 stub이 안 된다고 비즈니스 제약 조건을 함부로 허물면 안됑~

인터넷을 뒤져보니 아래와 같이 null을 특정 타입으로 강제 캐스팅하는 걸 이용하는 꼼수를 발견했다.

아래와 같이 `any()`를 만들어서 사용하면 Null 관련 에러가 발생하지 않는다.

```kotlin
@Suppress("UNCHECKED_CAST")
private fun <T> any(): T {
    Mockito.any<T>()
    return null as T
}
```

이제 아래와 같이 원래대로 Non Null로 해도 테스트 실행 에러가 발생하지 않는다.

```kotlin
fun findAll(pageable: Pageable): Page<WorldOut> {
    return worldRepository.findAll(pageable).map { WorldOut.fromEntity(it) }
}

given(worldQuery.findAll(any()))
    .willReturn(PageImpl(contents, PageRequest.of(page, size), total))
```

근데 이건 말 그대로 아무 값을 가진 객체가 와도 stub을 하는 거니까, `any()`보다는 `argThat()`을 사용하고 싶다.

꼼수를 응용해서 아래와 같이 간단하게 만들 수 있다. 물론 private으로 하지 말고 테스트 어디에서도 쓸 수 있게 하는 것이 좋다.

```kotlin
@Suppress("UNCHECKED_CAST")
private fun <T> argMatching(matcher: ArgumentMatcher<T>): T {
    argThat(matcher)
    return null as T
}
```

