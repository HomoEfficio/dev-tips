# Project Loom in IntelliJ IDEA Conf 2022

2022년 10월에 있었던 IntelliJ IDEA Conf 2022의 Project Loom 세션 메모

- 원본 영상: https://www.youtube.com/watch?v=0DUlUzqr09I
- 발표 자료: https://nurkiewicz.com/slides/loom

## 대충 메모

- Virtual Thread: JVM 가상 스레드, 경량 스레드, 유저 스레드
- Carrier Thread: 기존 스레드, 커널이 인지하는 스레드
- Contiuation: 스스로 멈추고 멈춘 지점에서 다시 시작
- Java 19 Preview 에 포함
- 가상 스레드는 객체나 람다 하나 만드는 것과 같아서 수십만, 수백만 생성 가능
- **JVM 안에 별도의 가상 스레드 스케줄러가 있고, 이 스케줄러가 가상 스레드 중에 CPU 사용이 필요한 가상 스레드를 선별해서 캐리어 스레드에 할당**
- 가상 스레드가 file io 등 블로킹 코드를 만나면 실행을 멈추고 yield
- yield 되면
  - 스케줄러가 가상 스레드를 캐리어 스레드에서 빼내고 다른 가상 스레드 할당
  - 가상 스레드에 있던 스택 트레이스 등 정보는 힙으로 이동
  - 결국 가상 스레드에서 블로킹 코드를 호출해도 캐리어 스레드를 점유하지 않고
  - 스택 트레이스 정보도 힙에서 다시 가져와서 진행하므로 컨텍스트 스위칭 비용도 적음
- 블로킹 호출이 많을 수록 많은 가상 스레드를 사용할 수 있음
  - 반대로 CPU 바운드 작업이 많을 수록 가상 스레드 사용 효과는 적음
- 하지만 힙을 사용하므로 GC 발생 빈도 증가, 스택트레이스가 깊은 경우 악영향
- 가볍다고 막 쓰면 연동된 다른 시스템에 과부하 전파 우려
  - 적절한 스로틀링 필요
- 리액터의 자원 효율성은 대체할 수 있지만 백프레셔를 대체하지는 못함

## 스프링은 어떻게 지원할까?

[여기](https://spring.io/blog/2022/10/11/embracing-virtual-threads) 나온 내용을 보면 대략 다음과 같이 기존 `AsyncTaskExecutor`를 통해 사용할 수 있을 것 같다.

```java
@Bean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
public AsyncTaskExecutor asyncTaskExecutor() {
  return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}

@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
  return protocolHandler -> {
    protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
  };
}
```

