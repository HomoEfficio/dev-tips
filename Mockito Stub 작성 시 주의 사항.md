# Mockito Stub 작성 시 주의 사항

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

