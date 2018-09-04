# Mockito when-thenReturn 사용 시 WrongTypeOfReturnValue 오류

다음과 같이 작성된 Mockito 코드에서 

```java
Mockito.when(this.modelRepository.save(model))
                .thenReturn(model);
```

다음과 같은 오류가 난다.

```java
org.mockito.exceptions.misusing.WrongTypeOfReturnValue: 
StatsModel cannot be returned by toString()
toString() should return String
***
If you're unsure why you're getting above error read on.
Due to the nature of the syntax above problem might occur because:
1. This exception *might* occur in wrongly written multi-threaded tests.
   Please refer to Mockito FAQ on limitations of concurrency testing.
2. A spy is stubbed using when(spy.foo()).then() syntax. It is safer to stub spies - 
   - with doReturn|Throw() family of methods. More in javadocs for Mockito.spy() method.
```

물론 `model`은 String 타입이 아님에도 위와 같은 에러가 난다.

에러 메시지를 읽어보면 String 관련 내용은 해당 사항 없어 보이고, 멀티 스레드를 사용하는 코드도 아니다.

`doReturn|Throw()` 를 사용하는 것이 더 안전하다는 내용에 따라 다음과 작성하면 에러가 나지 않는다.

```java
doReturn(model).when(this.modelRepository).save(model);
```

이렇게 `doReturn()` 방식으로 실행한 후에 다시 위에 있는 `when-thenReturn`을 사용하면 또 에러가 발생하지 않는다.

아쉽지만 이유는 잘 모르겠고, 테스트 코드니까 그냥 되는 방식으로 작성하기로..
