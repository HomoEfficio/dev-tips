# Java/Spring Asynchronous Programming 간단 정리

## Java 스레드 풀 관리 및 실행

### Executor

- Java 5 에서 도입
- 단순히 `void execute(Runnable command)`만 포함하고 있는 가장 기본 인터페이스

### ExecutorService

- Java 5 에서 도입
- `Executor`를 상속한 인터페이스로서 Future를 반환하는 반환값을 가지는 `submit()`와 `invokeAny()`, `invokeAll()`, `shutdown()`, `shutdownNow()`, `awaitTermination()` 등 포함

### ThreadPoolExecutor

- Java 5 에서 도입
- `ExecutorService`의 실질적인 구현체
- 스레드 풀 관리를 위한 다양한 기능 포함

### Executors

- Java 5 에서 도입
- static 메서드로 `ForkJoinPool` 또는 `ThreadPoolExecutor` 기반의 다양한 `ExecutorService` 구현체 생성 기능 제공


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
- ListenableFuture를 반환하는 `submitListenable()` 포함

### ThreadPoolTaskExecutor

- 스프링 2.0 에서 도입
- `ThreadPoolExecutor`를 쉽게 만들어 사용할 수 있게 해주는 일종의 Wrapper로서, 실제 스레드 풀 관리 역할은 `ThreadPoolTaskExecutor`의 인스턴스 변수로 포함되어 있는 JDK의 `ThreadPoolExecutor`가 담당

### @EnableAsync

- 애플리케이션에 기본적인 Async 기능 부여
- `@EnableAsync`가 설정되어 있지 않으면 `@Async`가 붙어있는 메서드 또는 `@Async`가 붙어있는 클래스의 메서드가 별도의 스레드에서 실행되지 않고 기존 스레드에서 실행되어 결과적으로 Async 효과가 발생하지 않는다.

### @Async

- 애플리케이션에 `@EnableAsync`가 설정되어 있어야 효과 발생
- 클래스나 메서드에 붙일 수 있다.
- 클래스에 붙이면 해당 클래스내의 모든 메서드에 @Async 효과 적용
- @Async("nameOfTaskExecutor")와 같이 `TaskExecutor`의 이름을 명시하면 그 `TaskExecutor`에서 스레드를 할당 받고, 할당 받은 스레드에서 `@Async`가 붙은 메서드를 실행한다.
- `@Async`와 같이 `TaskExecutor`의 이름을 명시하지 않으면
    - `TaskExecutor` 타입의 Bean을 찾아서 있으면 그 `TaskExecutor`에서 스레드를 할당 받아서 메서드 실행
    - `TaskExecutor` 타입의 Bean이 하나 이상이면서 `@Primary`도 지정이 안 되어있으면 다음의 INFO 메시지와 함께 `SimpleAsyncTaskExecutor`에서 스레드를 할당 받아서 메서드 실행
    - `TaskExecutor` 타입의 Bean이 없으면 `SimpleAsyncTaskExecutor`에서 스레드를 할당 받아서 메서드 실행

## Java의 Async Task Result Container

### Future

### CompletableFuture

## Spring의 Async Task Result Container

### ListenableFuture

### DeferredResult



