# Mockito Stub 작성 시 주의 사항

## eq() 의 적절한 사용 

아래와 같이 Stubbing하는 테스트를 실행하면 

```java
... 생략
        List<Long> someIds = Arrays.asList(63L, 77L, 82L, 91L);
        List<String> yyyyMMdds = Arrays.asList("20181101", "20181102", "20181103");
        given(this.someRepository
                .findSomethingWith(someIds, eq(yyyyMMdds)))
                .willReturn(someTestData);
... 생략
```

다음과 같은 에러가 발생한다.

```
org.mockito.exceptions.misusing.InvalidUseOfMatchersException: 
Invalid use of argument matchers!
2 matchers expected, 1 recorded:
-> at 테스트 코드 위치

This exception may occur if matchers are combined with raw values:
    //incorrect:
    someMethod(anyObject(), "raw String");
When using matchers, all arguments have to be provided by matchers.
For example:
    //correct:
    someMethod(anyObject(), eq("String by matcher"));

For more info see javadoc for Matchers class.
```

많이 헤멨는데 단서는 아래 메시지에 있었다.

```
2 matchers expected, 1 recorded:
```

2개의 Matcher를 예상했는데 1개만 기록되었다는 얘기다.

그래서 다음과 같이 Stubbing 할 메서드의 모든 파라미터를 Matcher를 통해 전달해주면 에러가 발생하지 않는다.

```java
... 생략
        List<Long> someIds = Arrays.asList(63L, 77L, 82L, 91L);
        List<String> yyyyMMdds = Arrays.asList("20181101", "20181102", "20181103");
        given(this.someRepository
                .findSomethingWith(eq(someIds), eq(yyyyMMdds)))  // <- someIds를 eq(someIds)로 변경
                .willReturn(someTestData);
... 생략
```

## doReturn-when-method 활용

간혹 아래와 같이 Stubbing을 하면

```java
given(this.userRepository.findByEmail(eq("hanmomhanda@gmail.com")))
                .willReturn(Optional.of(getUser()));
```

아래와 같은 에러가 난다.

```
org.mockito.exceptions.misusing.WrongTypeOfReturnValue: 
Optional cannot be returned by toString()
toString() should return String
***
If you're unsure why you're getting above error read on.
Due to the nature of the syntax above problem might occur because:
1. This exception *might* occur in wrongly written multi-threaded tests.
   Please refer to Mockito FAQ on limitations of concurrency testing.
2. A spy is stubbed using when(spy.foo()).then() syntax. It is safer to stub spies - 
   - with doReturn|Throw() family of methods. More in javadocs for Mockito.spy() method.
```

정확하진 않고 늘 되는지도 모르지만 이럴 때는 given-when-then 을 쓰지 말고 아래와 같이 doReturn-when-method 을 쓰면 된다.

```java
doReturn(Optional.of(getUser()))
    .when(this.userRepository).findByEmail("hanmomhanda@gmail.com");
```

심지어 이렇게 한 번 doReturn-when-method 을 써서 해결한 후 doReturn-when을 지우고 given-when-then을 쓰면 되기도 한다.

## 반환 값에 또 Mock이 사용되지 않도록 유의

아래와 같은 테스트 코드를 실행해서

```java
when(reviewService.toReview(any())
    .thenReturn(Mono.just(objectMapper.readValue(reviewVmJson, ReviewVM.class)))
```

아래와 같은 에러가 나왔다.

```
Unfinished stubbing detected here:
-> at com.nhaarman.mockitokotlin2.BDDMockitoKt.given(BDDMockito.kt:37)

E.g. thenReturn() may be missing.
Examples of correct stubbing:
    when(mock.isOk()).thenReturn(true);
    when(mock.isOk()).thenThrow(exception);
    doThrow(exception).when(mock).someVoidMethod();
Hints:
 1. missing thenReturn()
 2. you are trying to stub a final method, which is not supported
 3. you are stubbing the behaviour of another mock inside before 'thenReturn' instruction is completed

org.mockito.exceptions.misusing.UnfinishedStubbingException: 
Unfinished stubbing detected here:
-> at com.nhaarman.mockitokotlin2.BDDMockitoKt.given(BDDMockito.kt:37)

E.g. thenReturn() may be missing.
Examples of correct stubbing:
    when(mock.isOk()).thenReturn(true);
    when(mock.isOk()).thenThrow(exception);
    doThrow(exception).when(mock).someVoidMethod();
Hints:
 1. missing thenReturn()
 2. you are trying to stub a final method, which is not supported
 3. you are stubbing the behaviour of another mock inside before 'thenReturn' instruction is completed
```

3번 메시지가 해석이 좀 어려운데, 쉽게 말해 `thenReturn()` 안에서도 또 mock이 사용되고 있다는 얘기다.

위 테스트 코드에는 안 나와있지만, objectMapper가 `@MockBean private ObjectMapper objectMapper`로 선언돼 있었다능..  
그래서 `thenReturn()` 안에서 mock 객체로 선언된 objectMapper가 사용되기 때문에 위와 같은 에러가 발생한 것이다.
