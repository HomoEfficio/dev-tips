# Resilience4j-Bulkhead

## Bulkhead

Bulkhead(격벽)은 배의 선실 내에 물이 들어올 때 선실 전체에 물이 차지 않도록 여러 구역으로 나눠 놓은 것을 의미한다.  
시스템에서도 마찬가지로 스레드를 제때 반환하지 않는 문제 있는 api 호출로 시스템 전체 스레드가 고갈되지 않도록 일부 스레드만 배분하는 것을 의미한다.

### Semaphore Bulkhead

외부 api 호출하기 전 원래 실행되던 스레드에서 이어서 외부 api를 호출하되 외부 api를 호출할 수 있는 스레드의 갯수를 조절할 수 있는 **Semaphore Bulkhead가 기본**이고,  
원래 실행되던 스레드가 아니라 **별도의 스레드풀에 있는 스레드를 통해 외부 api를 호출할 수 있는 ThreadPoolBulkhead도 있다.**

### Threadpool Bulkhead

ThreadPoolBulkhead를 사용하면 별도의 스레드에서 외부 api를 호출하므로 원래 실행되던 스레드의 ThreadLocal에 있던 값이 외부 api를 호출하는 별도의 스레드에는 전달되지 않는다. 그래서 스프링에서는 ThreadLocal을 사용하는 SecurityContextHolder나 @Transactional이 제대로 동작하지 않게 된다.

하지만 이런 단점도 다음과 같은 Context Propagator를 사용해서 보완할 수 있다.

```java
public interface ContextPropagator<T> {

    Supplier<Optional<T>> retrieve();

    Consumer<Optional<T>> copy();

    Consumer<Optional<T>> clear();

    ...
```

ContextProgagator 인터페이스 구현체를 만들어서 아래와 같이 ThreadPoolBulkheadConfig 생성 시 지정해주면 된다.

```kotlin
val config = ThreadPoolBulkheadConfig.custom()
  .maxThreadPoolSize(2)
  .coreThreadPoolSize(1)
  .queueCapacity(1)
  .contextPropagator(new MyContextPropagator())  // 여기!!
  .build()
```




