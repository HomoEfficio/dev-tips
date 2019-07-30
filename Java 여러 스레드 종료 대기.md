# Java 여러 스레드의 종료 대기


## CompletableFuture 사용

```java
List<CompletableFuture<Void>> cFutures = new ArrayList<>();

for (...) {
    Completable<Void> cFuture = CompletableFuture.runAsync(new XXXRunnable(...), threadPoolTaskExecutor);
    cFutures.add(cFuture);
}

CompletableFuture<Void> allCFutures = CompletableFuture.allOf(cfutures.toArray(new CompletableFuture[0]));

allCFutures.get();
```

종료 대기만을 위해서 하는 작업치고는 뭔가 좀 번잡스러워 보인다.

## CountdownLatch

```java
CountdownLatch countdownLatch = new CountdownLatch(N);

for (...) {
    threadPoolTaskExecutor.execute(new XXXRunnable(..., countdownLatch));
}

countdownLatch.await();
```

모든 스레드 종료를 대기한다는 의도를 코드에서 그대로 읽을 수 있다.


## 그냥 join() 사용

검색해보면 대략 아래와 같은 방식을 소개하는 경우도 있는데, 이 방법은 쓰지 않는 게 좋다.

```java
List<Thread> threads = new ArrayList<>();

for (...) {
    threadN.start();
    threads.add(threadN);
}

for (...) {
    threadN.join();
}
```
이건 CompletableFuture 방식이나 CountdownLatch 방식이랑 비슷해보이지만 실제로는 확연히 다르다.  
왜냐하면 threadN.join() 호출 후에는 threadN이 종료될 때까지 메인스레드가 대기 상태로 되고,  
그 대기 상태로 남아있는 동안 threadNplus1.join()이 호출되지 않으므로 threadNplus1 종료를 기다리지 않고 메인스레드로 되돌아 갈 수 있기 때문이다.  
따라서 **이 방법은 의도와 다르게 동작할 수 있으며 사용하지 않는 것이 좋다.**

## 정리

`exceptionally`로 예외처리를 할 게 아니라면, 여러 스레드 모두 종료될 때까지 대기할 때는 `CountdownLatch`를 쓰는 쪽이 더 나은 것 같다.
