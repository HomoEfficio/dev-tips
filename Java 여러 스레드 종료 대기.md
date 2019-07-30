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

이 방식도 결과적으로는 모든 스레드 종료 때까지 대기하긴 하지만, `join()`이 메인 스레드를 블로킹한다는 걸 생각하면 조금 생각을 해보게 된다.
예를 들어 실행 시간이 각각 3, 2, 1초 걸리는 스레드 a, b, c가 있다고 하면, `a.join()` 호출 후 메인 스레드가 블로킹되는 동안 `b.join()`과 `c.join()`은 호출되지 않은 채로 b, c 스레드는 종료한다. 그리고 종료된 후에 메인 스레드에서 `b.join()`, `c.join()`가 순서대로 호출된다.

종료된 스레드에 대해 `join()`을 호출하면 에러 나는 거 아닌가? 싶지만 [여기](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/lang/Thread.java#l1233)를 보면 다행히도 그냥 아무일도 발생하지 않게 구현돼 있다.

실행 시간이 3, 2, 1이 아니라 1, 2, 3 또는 2, 1, 3 또는 1, 3, 2 등 그 외 모든 경우에도 결과적으로는 모든 스레드 종료 때까지 대기하긴 한다.  
하지만 `join()`이 블로킹을 유발한다는 점을 생각해보면 `join()`을 여러번 호출하는 것은 위와 같은 불필요한 생각 오버헤드를 발생시킨다.

## 정리

`exceptionally`로 예외처리를 할 게 아니라면, 여러 스레드 모두 종료될 때까지 대기할 때는 `CountdownLatch`를 쓰는 쪽이 제일 나은 것 같다.
