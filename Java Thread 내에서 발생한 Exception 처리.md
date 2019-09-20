# Java Thread 내에서 발생한 Exception 처리

# main 스레드에서 발생한 예외

보통 `try-catch` 내부에서 발생한 예외는 `catch` 문에서 잡아서 처리할 수 있다.

```java
public class TryCatchMainThread {

    public static void main(String[] args) {
        try {
            String simpleName = TryCatchMainThread.class.getSimpleName();
            System.out.println("OOO " + simpleName + " 실행 OOO");
            if (true) throw new NullPointerException("XXXXX " + simpleName + " 예외 발생 XXXXX");
        } catch (Exception e) {
            throw new RuntimeException("NPE를 RuntimeException으로 감싸서 던지기", e);
        }
    }
}

// 결과
OOO TryCatchMainThread 실행 OOO
Exception in thread "main" java.lang.RuntimeException: NPE를 RuntimeException으로 감싸서 던지기
    at io.homo_efficio.scratchpad.thread.exception.runner.TryCatchMainThread.main(TryCatchMainThread.java:15)
Caused by: java.lang.NullPointerException: XXXXX TryCatchMainThread 예외 발생 XXXXX
    at io.homo_efficio.scratchpad.thread.exception.runner.TryCatchMainThread.main(TryCatchMainThread.java:13)
```

이렇게 main 스레드에서 발생하는 예외 말고, main 스레드가 아닌 별도의 스레드 내에서 발생한 예외도 이 방식으로 잡아서 처리할 수 있을까?


# 별도의 스레드에서 발생한 예외

다음과 같이 `NullPointerException`을 발생시키는 `Runnable` 구현체를 별도의 스레드에서 실행해보자.

```java
public class ExceptionProducingRunnable implements Runnable {
    @Override
    public void run() {
        String simpleName = this.getClass().getSimpleName();
        System.out.println("OOO " + simpleName + " 실행 OOO");
        if (true)
            throw new NullPointerException(
                    "XXXXX " + simpleName + " 예외 발생 in thread [" + Thread.currentThread().getName() + "]");
    }
}
```

## 1. Thread.start(), try-catch 없음

```java
public class ThreadRunnerWithoutTryCatch {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new ExceptionProducingRunnable());
        thread.start();
        thread.join();
        System.out.println(ExecutorServiceUtils.OOO_MAIN_THREAD_정상_종료_OOO);
    }
}

// 결과
OOO ExceptionProducingRunnable 실행 OOO
Exception in thread "Thread-0" java.lang.NullPointerException: XXXXX ExceptionProducingRunnable 예외 발생 in thread [Thread-0]
    at io.homo_efficio.scratchpad.thread.exception.runnable.ExceptionProducingRunnable.run(ExceptionProducingRunnable.java:14)
    at java.base/java.lang.Thread.run(Thread.java:844)
OOO MAIN THREAD 정상 종료 OOO
```

별도의 스레드인 Thread-0 에서 예외가 발생한 것을 StackTrace에서 확인할 수 있지만, **별도의 스레드에서 발생한 예외는 무시되고 main 스레드가 정상 종료된다.**


## 2. Thread.start(), try-catch 있음

```java
public class ThreadRunnerWithTryCatch {

    public static void main(String[] args) throws InterruptedException {
        try {
            Thread thread = new Thread(new ExceptionProducingRunnable());
            thread.start();
            thread.join();
            System.out.println(OOO_MAIN_THREAD_정상_종료_OOO);
        } catch (Exception e) {
            System.out.println(CCC_THREAD_내에서_발생한_예외_CATCH_CCC);
            throw e;
        }
    }
}

// 결과
OOO ExceptionProducingRunnable 실행 OOO
Exception in thread "Thread-0" java.lang.NullPointerException: XXXXX ExceptionProducingRunnable 예외 발생 in thread [Thread-0]
    at io.homo_efficio.scratchpad.thread.exception.runnable.ExceptionProducingRunnable.run(ExceptionProducingRunnable.java:14)
    at java.base/java.lang.Thread.run(Thread.java:844)
OOO MAIN THREAD 정상 종료 OOO
```

마찬가지로 별도의 스레드인 Thread-0 에서 예외가 발생한 것을 StackTrace에서 확인할 수 있다.  
그런데 main 스레드에 try-catch 있음에도 불구하고, **별도의 스레드에서 발생한 예외는 try-catch가 없을 때와 마찬가지로 잡히지 않고 main 스레드가 정상 종료된다.**

`ExecutorService.execute()`로 실행해보면 어떨까?


## 3. ExecutorService.execute()

```java
public class ExecutorExecuteWithTryCatch {

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = null;
        try {
            executorService = Executors.newCachedThreadPool();
            executorService.execute(new ExceptionProducingRunnable());
            executorService.awaitTermination(1, TimeUnit.SECONDS);
            System.out.println(OOO_MAIN_THREAD_정상_종료_OOO);
        } catch (Exception e) {
            System.out.println(CCC_THREAD_내에서_발생한_예외_CATCH_CCC);
            throw e;
        } finally {
            if (Objects.nonNull(executorService))
                ExecutorServiceUtils.shutdownAndAwaitTermination(executorService);
        }
    }
}

// 결과
OOO ExceptionProducingRunnable 실행 OOO
Exception in thread "pool-1-thread-1" java.lang.NullPointerException: XXXXX ExceptionProducingRunnable 예외 발생 in thread [pool-1-thread-1]
    at io.homo_efficio.scratchpad.thread.exception.runnable.ExceptionProducingRunnable.run(ExceptionProducingRunnable.java:14)
    at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
    at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    at java.base/java.lang.Thread.run(Thread.java:844)
OOO MAIN THREAD 정상 종료 OOO
```

`ExecutorService.execute()`를 사용해보니 `ThreadPoolExecutor` 관련 StackTrace가 추가되긴 했지만, **별도의 스레드에서 발생한 예외가 catch 되지 못하고 main 스레드가 정상 종료된 것은 `thread.start()`로 실행했을 때와 같다.**

이쯤에서 알 수 있는 것은 **별도의 스레드 내에서 발생하는 예외는 특별한 방법을 사용하지 않으면 main 스레드에서 잡아서 처리할 수 없다**는 점이다.

그럼 어떻게 해야 별도의 스레드 내에서 발생한 예외를 잡아서 처리할 수 있을까?


# 별도의 스레드 내에서 발생한 예외 잡는 방법

## 1. ExecutorService.submit() + Future.get()

```java
public class ExecutorSubmitAndGetWithTryCatch {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executorService = null;
        try {
            executorService = Executors.newCachedThreadPool();
            Future<?> future = executorService.submit(new ExceptionProducingRunnable());
            Object result = future.get();
            System.out.println(OOO_MAIN_THREAD_정상_종료_OOO);
        } catch (Exception e) {
            System.out.println(CCC_THREAD_내에서_발생한_예외_CATCH_CCC);
            throw e;
        } finally {
            if (Objects.nonNull(executorService))
                shutdownAndAwaitTermination(executorService);
        }
    }
}

// 결과
OOO ExceptionProducingRunnable 실행 OOO
CCC Thread 내에서 발생한 예외 catch CCC
Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.NullPointerException: XXXXX ExceptionProducingRunnable 예외 발생 in thread [pool-1-thread-1]
    at java.base/java.util.concurrent.FutureTask.report(FutureTask.java:122)
    at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:191)
    at io.homo_efficio.scratchpad.thread.exception.runner.ExecutorSubmitAndGetWithTryCatch.main(ExecutorSubmitAndGetWithTryCatch.java:21)
Caused by: java.lang.NullPointerException: XXXXX ExceptionProducingRunnable 예외 발생 in thread [pool-1-thread-1]
    at io.homo_efficio.scratchpad.thread.exception.runnable.ExceptionProducingRunnable.run(ExceptionProducingRunnable.java:14)
    at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:514)
    at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
    at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
    at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    at java.base/java.lang.Thread.run(Thread.java:844)
```

`ExecutorService.execute()` 대신 **`ExecutorService.submit()`로 실행해서 별도의 스레드에서 실행한 결과를 `Future`에 담고, `Future.get()`으로 결과를 가져오면 드디어 별도의 스레드에서 발생한 예외가 main 스레드에서 잡힌다.** catch에서 `throw e`로 예외를 던졌으므로 StackTrace가 출력되고 main 스레드는 정상적으로 종료되지 못한다. `throw e`가 없으면 StackTrace도 출력되지 않고 main 스레드는 정상 종료 된다.

StackTrace를 살펴보면 별도의 스레드 내에서 발생한 예외는 `NullPointerException`이었지만, main 스레드에서 잡힌 예외는 `ExecutionException`다. 즉, **별도의 스레드에서 `NullPointerException`이 잡히고 `ExecutionException`으로 감싸져서 다시 던져진 상태로 `Future`에 담겨져 있다가, main 스레드에서 `Future.get()`으로 결과를 가져올 때 `ExecutionException`이 꺼내져서 던져진다**는 것을 알 수 있다. 참고로 `throw e`로 던져지는 `ExecutionException`은 unchecked exception 이므로 별도의 try-catch가 필요 없다.

`Future`의 구현체인 `FutureTask` 소스를 보면 **`get()`에 의해서만 호출되는 `report()`가 별도의 스레드에서 발생한 예외를 `ExecutionException`으로 감싸서 던지고 있다. 즉, `get()`을 호출하지 않으면 `report()` 예외가 main 스레드로 던져지지 않는다.** 확인해보자.

### ExecutorService.submit()

```java
public class ExecutorSubmitWithTryCatch {

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = null;
        try {
            executorService = Executors.newCachedThreadPool();
            Future<?> future = executorService.submit(new ExceptionProducingRunnable());
//            Object result = future.get();
            executorService.awaitTermination(1, TimeUnit.SECONDS);
            System.out.println(OOO_MAIN_THREAD_정상_종료_OOO);
        } catch (Exception e) {
            System.out.println(CCC_THREAD_내에서_발생한_예외_CATCH_CCC);
            throw e;
        } finally {
            if (Objects.nonNull(executorService))
                shutdownAndAwaitTermination(executorService);
        }
    }
}

// 결과
OOO ExceptionProducingRunnable 실행 OOO
OOO MAIN THREAD 정상 종료 OOO
```

`ExecutorService.submit()` 호출 후 `Future.get()`을 호출하지 않으면 예상대로 예외가 main 스레드로 던져지지 않으며, 예외도 잡히지 않고 main 스레드는 정상 종료된다. 

**별도의 스레드 내에서 분명히 발생한 예외에 대한 StackTrace 조차 남지 않으므로 가장 안 좋은 시나리오**지만 실무에서는 `ExecutorService.submit()`을 쓰면서 `Future.get()`을 쓰지 않는 상황은 거의 없을 것이다. 혹시라도 있다면 예외 발생 시 StackTrace라도 찍히도록 `ExecutorService.execute()`로 바꾸는 편이 낫다.

어쨌든 `Future`를 통해서 가능하다면 `CompletableFuture`로도 가능하지 않을까?


## 2. CompletableFuture.exceptionally()

```java
public class CompletableFutureRunAsync {

    public static void main(String[] args) {
        CompletableFuture.runAsync(new ExceptionProducingRunnable())
                .exceptionally(t -> {
                    System.err.println("예외 처리 in thread " + Thread.currentThread().getName());
                    System.out.println(CCC_THREAD_내에서_발생한_예외_CATCH_CCC);
                    t.printStackTrace();
                    throw new RuntimeException("exceptionally에서 throw", t);
                });
        System.out.println(OOO_MAIN_THREAD_정상_종료_OOO);
    }
}

// 결과
OOO ExceptionProducingRunnable 실행 OOO
예외 처리 in thread main
CCC Thread 내에서 발생한 예외 catch CCC
java.util.concurrent.CompletionException: java.lang.NullPointerException: XXXXX ExceptionProducingRunnable 예외 발생 in thread [ForkJoinPool.commonPool-worker-1]
    at java.base/java.util.concurrent.CompletableFuture.encodeThrowable(CompletableFuture.java:314)
    at java.base/java.util.concurrent.CompletableFuture.completeThrowable(CompletableFuture.java:319)
    at java.base/java.util.concurrent.CompletableFuture$AsyncRun.run(CompletableFuture.java:1739)
    at java.base/java.util.concurrent.CompletableFuture$AsyncRun.exec(CompletableFuture.java:1728)
    at java.base/java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:283)
    at java.base/java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1603)
    at java.base/java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:175)
Caused by: java.lang.NullPointerException: XXXXX ExceptionProducingRunnable 예외 발생 in thread [ForkJoinPool.commonPool-worker-1]
    at io.homo_efficio.scratchpad.thread.exception.runnable.ExceptionProducingRunnable.run(ExceptionProducingRunnable.java:14)
    at java.base/java.util.concurrent.CompletableFuture$AsyncRun.run(CompletableFuture.java:1736)
    ... 4 more
OOO MAIN THREAD 정상 종료 OOO
```

`CompletableFuture.exceptionally()`도 별도의 스레드에서 발생한 예외를 잡아서 처리할 수 있다.

`CompletionException`으로 감싸서 던져진다는 것과, 예외 핸들러인 람다식 안에서 처리되므로 `printStackTrace()`를 명시적으로 호출하지 않으면 StackTrace가 찍히지 않는다는 것이 `Future.get()`과 다르다.

예외 핸들러 마지막에 `throw new RuntimeException("exceptionally에서 throw", t);`로 던져진 예외는 새로 생성되는 `CompletableFuture`에 담겨지지만 그 이후 아무런 예외 핸들러가 없으므로 처리되지 않은 채로 main 스레드는 정상 종료된다는 점도 다르다.

여기에서는 `CompletableFuture.execptionally()`만 다뤘지만, `CompletableFuture.whenComplete()`나 `CompletableFuture.handle()`로도 예외를 잡아서 처리할 수 있다.


## 3. 스레드의 예외 핸들러 등록

`CompletableFuture`를 포함해서 `Future`를 사용하는 방법 말고도 `UncaughtExceptionHandler`를 사용하는 방법도 있다. 이 방법은 http://hochulshin.com/java-multithreading-thread-exception-handling/ 여기에 아주 잘 정리돼있다.

요약하면 다음과 같이 모든 스레드에 디폴트 `UncaughtExceptionHandler`를 등록할 수도 있고, 

```java
Thread.setDefaultUncaughtExceptionHandler(
        (thread, throwable) -> System.out.println(
                String.format(
                        "Default: [%s] 스레드에서 발생한 [%s] 처리",
                        thread.getName(), throwable.getMessage()))
);
Thread aThread = new Thread(() -> {
    throw new NullPointerException("널포인터 예외 A");
});
aThread.start();

// 결과
Default: [Thread-0] 스레드에서 발생한 [널포인터 예외 A] 처리
```

다음과 같이 스레드 별로 다른 `UncaughtExceptionHandler`를 등록해서 디폴트 핸들러를 덮어쓸 수도 있다.

```java
Thread bThread = new Thread(() -> {
    throw new NullPointerException("널포인터 예외 B");
});
bThread.setUncaughtExceptionHandler(
        (thread, throwable) -> System.out.println(
                String.format(
                        "Custom: [%s] 스레드에서 발생한 [%s] 처리",
                        thread.getName(), throwable.getMessage()))
);
bThread.start();

// 결과
Custom: [Thread-1] 스레드에서 발생한 [널포인터 예외 B] 처리
```


# 정리

>- 별도의 스레드에서 발생한 예외는 main 스레드에서 잡히지 않는다.
>
>- 별도의 스레드에서 발생한 예외를 main 스레드에서 잡으려면 크게 3가지 방법이 있다.
>
>1. `Future.get()`을 사용한다.  
>1. `CompletableFuture`의 예외 처리 가능 메서드들(`exceptionally()`, `whenComplete()`, `handle()` 등)을 사용한다.  
>1. `UncaughtExceptionHandler` 구현체를 스레드 핸들러로 등록한다.

관련 소스는 https://github.com/HomoEfficio/java-thread-scratchpad 에서 확인할 수 있다.
