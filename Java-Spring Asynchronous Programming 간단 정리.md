# Java/Spring Asynchronous Programming

>Blocking/Nonblocking 과 Sync/Async 는 http://homoefficio.github.io/2017/02/19/Blocking-NonBlocking-Synchronous-Asynchronous/ 를 참고한다.

## Java 스레드 풀 관리 및 사용

### Executor

- Java 5 에서 도입
- 단순히 `void execute(Runnable command)`만 포함하고 있는 가장 기본 인터페이스
- 발음은 외국에서도 혼용하는 것으로 보여 딱히 정답은 없지만 https://www.merriam-webster.com/dictionary/executor 에 따르면 이그제큐터 보다는 엑시큐터 쪽이 더 나은 듯

### ExecutorService

- Java 5 에서 도입
- `Executor`를 상속한 인터페이스로서 Future를 반환하는 반환값을 가지는 `submit()`와 `invokeAny()`, `invokeAll()`, `shutdown()`, `shutdownNow()`, `awaitTermination()` 등 포함

### AbstractExecutorService

- Java 5 에서 도입
- `ExecutorService`의 일부 메서드에 대한 구현 포함

### ThreadPoolExecutor

- Java 5 에서 도입
- `AbstractExecutorService`를 상속하며 `ExecutorService`의 실질적인 구현체
- 스레드 풀 관리를 위한 다양한 기능 포함
- Thread는 기본값으로 `Executors.defaultThreadFactory()`에 의해 생성되며, 이렇게 생성된 스레드는 모두 동일한 ThreadGroup에 속하며 동일한 NORM_PRIORITY 우선 순위와 non-daemon status를 가진다.
- 주요 인스턴스 변수
    - `corePoolSize`: 풀의 기본 사이즈. 기본값 없음
    - `maxPoolSize`: 풀의 최대 사이즈. 기본값 없음
    - `keepAliveTime`: `corePoolSize`를 넘는 스레드나 `allowCoreThreadTimeOut`이 true 일 때 `corePoolSize` 이내의 idle 스레드는 `keepAliveTime`이 지나면 TimeOut 된다.
    - `allowCoreThreadTimeOut`: true이면 `corePoolSize`를 넘지 않는 스레드도 idle 상태이면 TimeOut 시킨다.
- Thread 생성 기본 전략
    - 풀의 활성화 된 스레드 수가 corePoolSize 보다 작은 상태에서 execute() 호출 시 스레드 새로 생성
    - 풀의 활성화 된 스레드 수가 corePoolSize와 maxPoolSize 사이에 있으면서 큐가 꽉 차있지 않으면 새 스레드를 생성하지 않고 태스크를 큐로 전송
    - 풀의 활성화 된 스레드 수가 corePoolSize와 maxPoolSize 사이에 있으면서 큐가 꽉 차있으면 스레드 새로 생성
    - 큐도 꽉 차있고 활성 스레드 수가 `maxPoolSize`에 도달해있는 상태에서 태스크가 또 들어오면 에러 발생
    - `corePoolSize`와 `maxPoolSize`를 같게 하면 결국 고정 크기 스레드 풀을 생성하게 된다.

### Executors

- Java 5 에서 도입
- static 메서드로 `ForkJoinPool` 또는 `ThreadPoolExecutor` 기반의 다양한 `ExecutorService` 구현체 생성 기능 제공하는 팩토리 클래스

### ForkJoinPool

- Java 7 에서 도입
- 풀에 있는 모든 스레드가 풀에 submit 된 태스크 또는 다른 태스크에 의해 생성된 태스크를 찾아서 실행하는 방식으로 노는 스레드를 최소화하는 work-stealing 개념 포함
- `ExecutoreService`는 트랜잭션 처리 같은 독립적인 태스크를 처리하는데 적합하고, `ForkJoinPool`은 작게 나누어 재귀적 실행으로 분할 정복 가능한 태스크를 처리하는데 적합
- `ForkJoinPool.commonPool()`로 공용 스레드 풀 쉽게 생성 가능
- `CompletableFuture`의 `***Async` 메서드 호출 시 별도의 `Executor`를 인자로 넘기지 않으면 기본값으로 `ForkJoinPool`에서 할당받은 스레드로 비동기 처리 

### ExecutorService의 종료 처리

- `ExecutorService`를 사용하는 애플리케이션 종료 시 `ExecutorService`의 우아한 종료를 위해 Java API 문서는 다음과 같은 코드를 권장한다.

    ```java
    void shutdownAndAwaitTermination(ExecutorService pool) {
      pool.shutdown(); // Disable new tasks from being submitted
      try {
        // Wait a while for existing tasks to terminate
        if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
          pool.shutdownNow(); // Cancel currently executing tasks
          // Wait a while for tasks to respond to being cancelled
          if (!pool.awaitTermination(60, TimeUnit.SECONDS))
            System.err.println("Pool did not terminate");
        }
      } catch (InterruptedException ie) {
        // (Re-)Cancel if current thread also interrupted
        pool.shutdownNow();
        // Preserve interrupt status
        Thread.currentThread().interrupt();
      }
    }
    ```


## Spring 스레드 풀 관리 및 실행

### TaskExecutor

- 스프링 2.0 에서 도입
- `Executor`를 상속한 인터페이스로서 `void execute(Runnable task)`를 오버라이드

### AsyncTaskExecutor

- 스프링 2.0.3 에서 도입
- `TaskExecutor`를 상속한 인터페이스
- `submit()`과 timeout 개념 추가

### AsyncListenableTaskExecutor

- 스프링 4.0 에서 도입
- `ListenableFuture`를 반환하는 `submitListenable()` 포함

### ThreadPoolTaskExecutor

- 스프링 2.0 에서 도입
- 자바의 `ThreadPoolExecutor`를 쉽게 만들어 사용할 수 있게 해주는 일종의 Wrapper로서, 실제 스레드 풀 관리 역할은 `ThreadPoolTaskExecutor`의 인스턴스 변수로 포함되어 있는 JDK의 `ThreadPoolExecutor`가 담당
- 주요 설정 변수 기본값
    - corePoolSize: 1
    - maxPoolSize: Integer.MAX_VALUE
    - keepAliveSeconds: 60
    - queueCapacity: Integer.MAX_VALUE
    - allowThreadTimeOut: false

### SimpleAsyncTaskExecutor

- 스프링 2.0 에서 도입
- `@Async` 애노테이션의 속성으로 `TaskExecutor`를 명시해주지 않으면 `SimpleAsyncTaskExecutor`가 기본으로 사용되며 스레드 이름이 `SimpleAsyncTaskExecutor-#`가 됨
- `@Async` 없이 `Callable<T>`를 반환하는 메서드가 사용되어도 `SimpleAsyncTaskExecutor`가 기본으로 사용되며 스레드 이름이 `MvcAsync#`가 됨
- 스레드를 재사용하지 않고 항상 새로 만들고 폐기하기 때문에 연습용 코드에나 적당하며, 실제 애플리케이션에서는 사용하지 않는 것이 좋다.
- 특히 많은 수의 금방 완료되는 태스크를 실행할 때는 이 `SimpleAsyncTaskExecutore`를 사용하지 말고 별도의 스레드 풀을 사용하는 것이 좋다.

### @EnableAsync

- 애플리케이션에 기본적인 Async 기능 부여
- `@EnableAsync`가 설정되어 있지 않으면 `@Async`가 붙어있는 메서드 또는 `@Async`가 붙어있는 클래스의 메서드가 별도의 스레드에서 실행되지 않고 기존 스레드에서 실행되어 결과적으로 Async 효과가 발생하지 않는다.

### @Async

- 애플리케이션에 `@EnableAsync`가 설정되어 있어야 효과 발생
- 클래스나 메서드에 붙일 수 있다.
- 클래스에 붙이면 해당 클래스내의 모든 메서드에 `@Async` 효과 적용
- `@Async(value = "nameOfTaskExecutor")`와 같이 `TaskExecutor` 빈(Bean)의 이름을 명시하면 그 `TaskExecutor`에서 스레드를 할당 받고, 할당 받은 스레드에서 `@Async`가 붙은 메서드를 실행한다.
- `@Async`와 같이 `TaskExecutor`의 이름을 명시하지 않으면
    - `TaskExecutor` 타입의 Bean을 찾아서 있으면 그 `TaskExecutor`에서 스레드를 할당 받아서 메서드 실행
    - `TaskExecutor` 타입의 Bean이 둘 이상이면서 `@Primary`도 지정이 안 되어 있어서 어느 `TaskExecutor`를 사용할 지 결정할 수 없으면 다음과 같은 INFO 메시지와 함께 `SimpleAsyncTaskExecutor`에서 스레드를 할당 받아서 메서드 실행
        
        ```
        2018-03-02 17:29:43.443  INFO 34224 --- [nio-8080-exec-1] .s.a.AnnotationAsyncExecutionInterceptor : More than one TaskExecutor bean found within the context, and none is named 'taskExecutor'. Mark one of them as primary or name it 'taskExecutor' (possibly as an alias) in order to use it for async processing: [threadTaskExecutor, tenThreadTaskExecutor]
        Thread name of @Async without name: SimpleAsyncTaskExecutor-1
        ```

    - `TaskExecutor` 타입의 Bean이 없으면 `SimpleAsyncTaskExecutor`에서 스레드를 할당 받아서 메서드 실행

### ThreadPoolTaskExecutor의 종료 처리

- `ThreadPoolTaskExecutor`는 `DisposableBean` 인터페이스를 구현하는 `ExecutorConfigurationSupport`를 상속하고 있어서, 스프링 애플리케이션이 종료될 때 `ExecutorConfigurationSupport.destroy()`가 자동으로 호출되면서 아래 로그와 같이 우아하게 종료되므로 개발자가 직접 별도의 종료 처리를 해주지 않아도 된다.

    ```
    2018-03-02 21:14:10.743  INFO 41729 --- [      Thread-10] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'tenThreadTaskExecutor'
    2018-03-02 21:14:10.744  INFO 41729 --- [      Thread-10] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'threadTaskExecutor'
    ```


## Java의 Async Task Result Container

### Future

### CompletableFuture

- Controller에서 바로 CompletableFuture를 반환하면 DeferredResult를 반환하는 것과 동일하게 동작하므로 DeferredResult를 대체할 수 있다.
- CompletableFuture는 여러 future의 조합을 쉽게 만들어주는 API를 제공해주고 자바 표준 API라는 면에서 DeferredResult 보다 장점이 있다.
- Blocking 모드로 오랜 시간 실행되는 메서드를 비동기로 실행하려면 `CompletableFuture.supplyAsync(Supplier, Executor)` 보다는 `@Async(ExecutorQualifierName) + CompletableFuture.completedFuture(result)`가 가독성이 더 좋은 코드를 짤 수 있다.

```java
    @Async("tenThreadTaskExecutor")
    public CompletableFuture<String> showFromCompletableFuture(int index) throws InterruptedException {
        System.out.println("Inside of @Async+CompletableFuture method, request index: " + index + ", thread name: " + Thread.currentThread().getName());
        Thread.sleep(2000);
        String info = "Result of @Async+CompletableFuture method, request index: " + index + ", thread name: " + Thread.currentThread().getName();
        System.out.println(info);
        return CompletableFuture.completedFuture(info);
    }
```

```java
    public CompletableFuture<String> returningCompletableFutureFromController(int index) {

        CompletableFuture<String> stringCompletableFuture = CompletableFuture.supplyAsync(
                () -> {
                    String result = "";
                    try {
                        result = this.normalService.showRawInfo(index);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    return result;
                },
                this.taskExecutor
        );

        System.out.println("Right after invoking Async Service without CompletableFuture with Request index: " + index + " in thread: " + Thread.currentThread().getName());

        return stringCompletableFuture;
    }
```


## Spring의 Async Task Result Container

### ListenableFuture

### DeferredResult



