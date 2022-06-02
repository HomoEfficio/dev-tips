# Kotlin Reactor switchIfEmpty

코틀린에서 리액터를 사용하다보면 값이 없는 상황에서의 처리를 위해 다음과 같이 `switchIfEmpty` 함수를 사용하게 된다.

```kotlin
xxxRepository.findAllByYyy(zzz)
    .switchIfEmpty(aaa)
    // 이하 생략
```

저렇게 사용하면 `findAllByYyy()` 의 결과값이 있든 없든 `aaa`는 항상 실행된다.

aaa가 예를 들어 `Mono.error(throwable)`처럼 그냥 인메모리 연산으로 끝나는 거라면 큰 차이 없다고 할 수도 있겠지만, 그게 아니라 DB나 외부 서비스를 호출하는 코드라면 자원 낭비 및 성능 손실 또는 의도하지 않은 오류로 이어질 수 있다.

이럴 때는 아래와 같이 람다식을 인자로 전달해주면 lazy 하게, 즉 진짜 `findAllByYyy()` 결과값이 없을 때만 실행되게 할 수 있다.

```kotlin
xxxRepository.findAllByYyy(zzz)
    .switchIfEmpty { aaa }
    // 이하 생략
```

이런 차이가 나는 이유는 `switchIfEmpty(aaa)`와 `switchIfEmpty { aaa }`는 다른 함수다.

`switchIfEmpty(aaa)`는 Reactor Core에 있는 함수이며 다음과 같이 정의돼 있다.

```java
public final Mono<T> switchIfEmpty(Mono<? extends T> alternate) {
		return onAssembly(new MonoSwitchIfEmpty<>(this, alternate));
	}
```

`switchIfEmpty { aaa }`는 Reactor Kotlin Extension에 있는 함수이며 다음과 같이 `Mono.defer`로 감싸서 lazy 실행되게 해주는 일종의 유틸 함수다.

```kotlin
fun <T> Mono<T>.switchIfEmpty(s: () -> Mono<T>): Mono<T> = this.switchIfEmpty(Mono.defer { s() })
```


따라서 **늘 `switchIfEmpty()`를 쓸 게 아니라 상황에 따라 `switchIfEmpty {}`를 사용해야 한다.**

그런데 `switchIfEmpty(aaa)`에 들어가는 것도 결국 Mono라서 실제 값이 없을 때만 aaa 가 subscribe 될테고, subscribe 될때만 aaa가 실행되므로 사실 상 lazy하게 처리되는 모양새라, 굳이 `switchIfEmpty {}`를 사용하지 않아도 되는 것 같기도.. 이건 더 조사가 필요.

참고로 자바의 Optional 에도 이와 비슷한 상황이 있으니 참고하자: https://github.com/HomoEfficio/dev-tips/blob/master/Java-Optional-바르게-쓰기.md#2-orelsenew--대신-orelseget---new-
