# Resilience4j-CircuitBreaker 실험

## circuit breaker

서킷 브레이커 관련 자료 대부분이 개념과 간단한 코드 예제는 잘 다루고 있지만, 여러가지 설정값이 구체적으로 어떤 효과를 발휘하는지 자세히 설명하는 자료가 별로 없어서 어쩔 수 없이 직접 해보고 정리해봤다.

개념적으로는 **OPEN 이 문제 있는 상태, CLOSED 가 문제 없는 상태라는 것만 용어에서 느껴지는 직관과 반대인 것만 주의**하면 되고, 그 외는 그냥 아래 두 가지 그림만으로도 충분하다고 생각한다.

![Imgur](https://i.imgur.com/Pt7tH33.jpg)

![Imgur](https://i.imgur.com/a9id8Cf.png)

출처: https://resilience4j.readme.io/docs/circuitbreaker

## 설정

아래와 같이 management 설정을 위와 같이 추가하면 `/actuator/health` 호출 결과에 circuit breaker 상태가 포함돼서 표시된다.

`instances` 바로 하위에 있는 `instanceA`은 코드에 적용할 서킷 브레이커 인스턴스 이름을 가리키며 `@CircuitBreaker(name = "instanceA")`와 같은 형식으로 사용된다.


```yml
resilience4j.circuitbreaker:
  configs:
    default:
      slidingWindowType: COUNT_BASED
      slidingWindowSize: 10
      permittedNumberOfCallsInHalfOpenState: 5
      waitDurationInOpenState: 60000
      failureRateThreshold: 50
      registerHealthIndicator: true
  instances:
    instanceA:
      baseConfig: default

management:
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true
```

## 적용

기본적으로는 `@CircuitBreaker(name = "instanceA", fallbackMethod = "fallbackForInstanceA")`와 같은 애너테이션을 붙이면 서킷 브레이커가 적용된다.

아래와 같이 `testForResilience4j()`, `testForOtherResilience4j()` 두 개의 메서드에 동일한 서킷 브레이커 인스턴스 이름을 지정하면, 두 메서드가 각각 호출될 때 `instanceA`라는 동일한 하나의 객체에 저장되는 `bufferedCalls`, `failedCalls` 등의 값이 변경된다. 즉 **서킷 브레이커 인스턴스 설정은 단순히 읽어서 사용할 불변값을 지정하기만 하는 일반적인 설정 정보와는 다르게 변경되는 상태를 저장하는 역할도 한다**는 특징이 있다.

설정하지 않은 인스턴스 이름을 지정해도, 예를 들어 `@CircuitBreaker(name = "XXYYZZ")`와 같이 사용해도 에러가 발생하지는 않는다. 다만 `XXYYZZ`라는 이름의 서킷 브레이커 인스턴스가 존재하지 않기 때문에 `bufferedCalls`, `failedCalls`가 저장될 곳이 없고, 따라서 OPEN, HALF_OPEN, CLOSED 같은 상태를 판별할 수 없으므로 결국 서킷 브레이커는 의도대로 동작하지 않는다.

```kotlin
    @CircuitBreaker(name = "instanceA", fallbackMethod = "fallbackForInstanceA")
    fun testForResilience4j(num: Int): String {

        if (num > 5) {
            throw RuntimeException("failed")
        }
        println("===== num: $num")

        return "CoreUserInfo $num"
    }

    private fun fallbackForInstanceA(num: Int, t: Throwable): String {
        t.printStackTrace()
        return "Fallback for InstanceA: $num"
    }

    @CircuitBreaker(name = "coreUserInfo", fallbackMethod = "fallbackForCoreUserInfo")
    fun testForOtherResilience4j(num: Int): String {

        if (num > 5) {
            throw RuntimeException("failed")
        }
        println("===== num: $num")

        return "CoreUserInfo $num"
    }
```

참고로 애너테이션을 통한 AOP 방식으로 서킷 브레이커를 적용할 때는 Bean 객체 안에 있는 메서드에 적용할 때만 서킷 브레이커가 동작하고, Bean 이 아닌 객체의 메서드에는 애너테이션을 붙여봐도 서킷 브레이커가 동작하지 않는다. 

또한 private 메서드에 애너테이션을 붙여봐도 서킷 브레이커가 동작하지 않는다.

애너테이션이 아닌 방식으로 직접 서킷 브레이커를 구현하는 방법은 https://reflectoring.io/circuitbreaker-with-resilience4j/ 여기에 잘 나와있다.

## 결론 먼저

### fallback 메서드

>- 서킷 브레이커를 적용 시 `fallbackMethod`를 지정했다면 서킷 브레이커가 적용된 메서드에서 예외 발생 시 발생된 예외가 fallback 메서드의 파라미터로 지정한 예외의 서브타입이면 서킷 브레이커의 상태(OPEN/HALF_OPEN/CLOSED)와 무관하게 fallback 메서드가 실행된다.
>- fallback 메서드는 여러 개일 수 있으며 예외 타입에 맞는 fallback 메서드가 실행된다.
>- OPEN 상태에서는 무조건 `io.github.resilience4j.circuitbreaker.CallNotPermittedException` 예외가 발생하므로, `CallNotPermittedException`를 파라미터에 포함한 fallback 메서드를 정의해두면 OPEN 상태에 대해 fallback 처리를 할 수 있다.

### 서킷 브레이커 진행 흐름

>1. 최초 CLOSED 상태에서 최초 요청이 들어오면 `bufferedCalls`가 1 증가하고, 요청이 실패하면 `failedCalls`도 1 증가한다.
>2. `bufferedCalls` 값이 `slidingWindowSize`에 도달하면 `failedCalls / bufferedCalls * 100%`를 통해 `failureRate`가 계산된다.  
>3. 계산 결과 `failureRateThreshold` 미만이면 CLOSED 상태가 유지되고,  
>3.1 이후 요청이 들어오면 최초에 들어왔던 요청이 버퍼에서 빠져나가면서 `bufferedCalls`가 1 감소하지만 새로 들어온 요청에 의해 1 증가해서 결국 10으로 그대로 유지된다.  
>3.2 빠져나간 요청이 성공한 요청이었다면 `failedCalls` 값도 그대로 유지되고, 빠져나간 요청이 실패한 요청이었다면 `failedCalls` 값도 1 감소한다. 그리고 새로 들어온 요청이 실패라면 `failedCalls` 값이 1 증가하고, 이 값을 기준으로 `failureRate` 값이 새로 계산된다. 계산 결과 `failureRateThreshold` 이하이면 3번 과정과 동일하다.
>4. 계산 결과 `failureRateThreshold` 이상이면 서킷 브레이커 상태가 OPEN 으로 변경되고,  
>4.1 OPEN 상태에서 `waitDurationInOpenState` 동안은 요청이 들어와도 외부 서비스를 호출하지 않고 `io.github.resilience4j.circuitbreaker.CallNotPermittedException`가 발생하며, 그래서 `bufferedCalls`도 `failedCalls`도 증가시키지 않고, `notPermittedCalls` 값만 1씩 증가한다.  
>4.2 `waitDurationInOpenState`이 지나서 요청이 들어오면 HALF_OPEN 으로 상태가 변경되고 10이었던 `bufferedCalls` 값이 1로 변경된다. 요청이 성공하면 `failedCalls`은 0, 실패하면 1로 변경된다.
>5. HALF_OPEN 상태에서 `bufferedCalls` 값이 `permittedNumberOfCallsInHalfOpenState`에 도달하면 `failureRate`가 계산되는데 이 값이,  
>5.1 `failureRateThreshold` 값 미만이면 상태가 CLOSED 로 변경되고 `bufferedCalls`, `failedCalls` 모두 0으로 초기화된다. 이후 과정은 1번 과정과 동일하다.  
>5.2 `failureRateThreshold` 값 이상이면 상태가 OPEN 으로 변경되고 `bufferedCalls`, `failedCalls` 값은 그대로 남는다. 이후 과정은 4번 과정과 동일하다.  


## 실험 내용

- COUNT_BASED 타입 기준으로 `slidingWindowSize: 10`이므로 최소 10개의 요청이 있어야 `failureRate`가 계산된다. 
  - 9개 요청까지는 실패 요청 수인 `failedCalls`값과 무관하게 `failureRate` 값은 `-1.0%` 인채로 유지된다.
- 10개의 요청이 들어와서 `failureRate` 값이 계산되면 `failureRateThreshold` 값과 비교해서 서킷 상태 변경이 발생할 수 있다.
  - `failureRateThreshold: 50`이므로 10개 중 5개 요청이 에러이면 즉 50%에 도달하면 `CIRCUIT_OPEN`으로 서킷 상태가 변경된다.
  - **[공식 문서](https://resilience4j.readme.io/docs/circuitbreaker)에는 `above` 즉 50%를 초과해야 OPEN 상태로 변경되는 것으로 나와있지만, 실제 테스트 결과 50% 초과가 아니라 50% 일 때도 OPEN 상태로 변경된다.**
- OPEN 됐을 때 주요 상태값은 다음과 같다.
  - state: OPEN
  - failureRate: 50%
  - bufferedCalls: 10
  - failedCalls: 5
  - notPermittedCalls: 0
- **OPEN 상태가 된 후 아무런 요청이 없으면 계속 OPEN 상태와 위 주요 상태값이 그대로 남아 있는다.**
- OPEN 상태에서 1번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 1
  - failedCalls: 1
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 2번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 2
  - failedCalls: 2
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 3번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 3
  - failedCalls: 3
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 4번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 4
  - failedCalls: 4
  - notPermittedCalls: 0
- **`permittedNumberOfCallsInHalfOpenState: 5`로 설정돼있으므로, HALF_OPEN 상태에서 5번째 요청이 에러를 발생시키면 `failureRate` 값이 계산되고 다음과 같이 주요 상태 값이 변경된다.**
  - state: OPEN
  - failureRate: 100.0%
  - bufferedCalls: 5
  - failedCalls: 5
  - notPermittedCalls: 0
- **OPEN 상태에서 `waitDurationInOpenState` 시간에 도달하기 전에 들어오는 요청은 성공/실패 무관하게 `notPermittedCalls`만 1씩 계속 증가시킨다.**
  - state: OPEN
  - failureRate: 100.0%
  - bufferedCalls: 5
  - failedCalls: 5
  - notPermittedCalls: 1
- **OPEN 상태에서 `waitDurationInOpenState` 시간이 지난 후에 1번째 요청이 들어오면 성공/실패 무관하게 HALF_OPEN 으로 상태가 바뀌고 정상 처리되면 다음과 같이 주요 상태 값이 변경된다.**
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 1
  - failedCalls: 0
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 2번째 요청이 정상 처리되면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 2
  - failedCalls: 0
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 3번째 요청이 정상 처리되면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 3
  - failedCalls: 0
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 4번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 4
  - failedCalls: 1
  - notPermittedCalls: 0
- **`permittedNumberOfCallsInHalfOpenState: 5`로 설정돼있으므로, HALF_OPEN 상태에서 5번째 요청이 에러를 발생시키면 `failureRate` 값이 계산되고(2/5 == 40%) 50% 미만이므로 CLOSED 상태가 되며 다음과 같이 주요 상태 값이 초기화된다.**
  - state: CLOSED
  - failureRate: -1.0%
  - bufferedCalls: 0
  - failedCalls: 0
  - notPermittedCalls: 0
