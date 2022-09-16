# Spring Reactor doOnExit(), doOnTerminate(), using()

### doOnExit()

- JVM이 정상 종료될 때만 발동
- https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/io/File.html#deleteOnExit()
- `tempFile.deleteOnExit()`과 같이 사용되면 서버가 오래 기동되면 파일이 삭제되지 않고 쌓여 나중에 Pod Eviction 에러로 이어질 수 있음


### doOnTerminate()

- complete, error 시 발동
- cancel 시에는 발동하지 않음
- https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#doOnTerminate-java.lang.Runnable-
- cancel 이 잘 발생하진 않지만, 많이 발생한다면 서버가 오래 기동될 수록 용량이 차서 Pod Eviction 에러로 이어질 수 있음


### using()

- try-with-resource와 유사하게 complte, error, cancel 모두에 대해 리소스 회수 처리하므로 뒷맛이 가장 개운하고 깔끔
- https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#using-java.util.concurrent.Callable-java.util.function.Function-java.util.function.Consumer-
- 사용법: https://medium.com/digitalfrontiers/reactive-patterns-try-catch-finally-dc2f212c42e1
