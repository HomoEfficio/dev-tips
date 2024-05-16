# Spring Reactor if 분기

리액터 플로우를 따라 처리하다보면 분기 상황에서 다음과 같이 `filter`, `switchIfEmpty`를 사용해서 처리한다.

```kotlin
externalApi.xxx(ppp)
    .filter { response -> response.isYYY() }
    .switchIfEmpty { Mono.error(zzz) }
    ...
```

그런데 이렇게 하면 `Mono.error()` 위치에서는 `response`에 접근할 수 없어 에러 로그에 적절한 정보를 담기 어렵다는 단점이 있다.

이럴 때는 다음과 같이 `handle`을 사용하면 더 많은 정보를 에러 처리에 사용할 수 있다.

```kotlin
externalApi.xxx(ppp)
    .handle<AaaResponse> { response, sink ->
        if (response -> response.isYYY()) {
            sink.next(response)
        } else {
            sink.error(<<<response 정보 사용 가능>>>)
        }
    }
```

