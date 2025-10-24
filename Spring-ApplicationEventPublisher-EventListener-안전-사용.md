# `@EventListener` 안전 사용

## 배경

- 느슨한 결합을 통해 불필요한 결합도 증가 없이 후행적인 로직을 붙일 수 있도록 스프링에서 `ApplicationEventPublisher`와 `@EventListener` 제공
- `@EventListener`는 누가 publisher인지 알지 못하고 알아서도 안 되고, publisher에 영향을 미쳐도 안됨
- 하지만 `@Async`등으로 비동기 적용하지 않은 동기식 `@EventListener`을 아래와 같이 사용할 때, `@EventListener`에서 예외가 발생하면 
    ```java
    class XxxService {
        // publisher 쪽 로직
        applicationEventPublisher.publishEvent(xxxEvent);
        // 나머지 로직
    }
    
    class XxxHandler {
        @EventListener
        public void handleXxx(XxxEvent) {
            // 이벤트 처리 로직
        }
    }
    ``` 
- `나머지 로직`이 실행되지 않을 수 있고, 동일한 이벤트를 처리하는 다른 listener도 호출되지 않을 수 있으므로, 결과적으로 publisher에게 영향이 전파됨
- 따라서 publisher 에게 영향이 없도록 하는 장치 필요
- 이를 위해 실무적으로 다음과 같이 명시적으로 try-catch를 적용하고 있으나
    ```java
	@EventListener
	public void handleXxx(Xxx xxx) {
	    try {
	        // 이벤트 처리 로직
	    } catch (Exception e) {
	        // 로깅 등 적절한 예외 처리
	    }
	}
    ```
- **휴먼 에러로 try-catch 적용 누락될 위험이 있어 더 근본적인 방지 장치 필요**

## Aspect 추가

- `@EventListener`를 처리하는 Aspect를 추가하여 try-catch로 감싸기 적용
- 이를 통해 `Spring Proxy -> @EventListener를 포함하는 Spring Bean -> @EventListener가 붙은 메서드` 흐름을
  - `Spring Proxy -> @EventListener Aspect(try-catch로 wrapping) -> @EventListener를 포함하는 Spring Bean -> @EventListener가 붙은 메서드`로 변경하며
- 결과적으로 `@EventListener`가 붙은 모든 메서드에 공통으로 try-catch 적용됨


## publisher 적절한 사용법

- 이벤트 기반 프로그래밍의 장점인 느슨한 결합이라 함은 publisher-listener 서로를 직접적으로 알지 못하게 간접화함을 의미하며,
- 이로 인한 장점은 결국 listener가 아무리 추가되더라도 pubilsher 쪽 코드가 변경될 필요가 없다는 점임
- 하지만 listener 추가에 따라 발행되는 이벤트의 타입을 계속 추가하면 publsher 쪽에도 `publishEvent()`를 계속 추가해야 하므로 publisher 쪽 코드도 계속 변경되므로 느슨한 결합의 장점이 사실상 사라짐
- 따라서 **특정 지점에서 발행되는 이벤트는 단 하나의 타입만 사용하는 것이 적절**
  - **listener를 새로 추가하면서 새로운 데이터가 필요해졌다면, 이벤트 타입을 새로 추가할 것이 아니라 기존의 이벤트 타입에 새 필드를 추가하는 것이 바람직**
  - 이는 꼭 `@EventListener`를 사용하기 위해 준수해야 하는 특별한 제약 사항이 아니라,
  - 하나의 행위(Behavior)의 완료에 따라 발생하는 이벤트(Event)에 대해서는 그에 따른 정보도 하나의 이벤트 타입으로 관리하는 것이 타당하며,
  - 하나의 타입 내에서 소화할 수 없는 상황이라면 그 행위와 이벤트의 관계가 적절한지 검토 필요

