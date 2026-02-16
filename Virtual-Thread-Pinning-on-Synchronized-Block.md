# Virtual Thread Pinning on Synchronized Block

Java 21에서 아래 `synchronized` 블록의 락을 가상 스레드 A가 획득한 상태에서,  
다른 가상 스레드 B가 `synchronized` 블록을 만나면 B는 락을 가지고 있지 않으므로 락 획득 시까지 기다려야 한다.

이 때 가상 스레드 B는 캐리어 스레드로부터 언마운트 될까 아니면 마운트 된 상태로 캐리어 스레드를 점유하며 자원을 낭비할까?

```java
synchronized (lock) {
    // 시간은 오래 걸리지만 blocking은 없는 사례
}
```

## 락 획득을 기다리는 동안에는 캐리어 스레드를 점유하지 않는다

아래의 코드로 확인해 볼 수 있다.

```java
public class VirtualThreadPinningExample {

    private static final Object lock = new Object();

    public static void main(String[] args) {

        giveSomeTimeForJFRInit();

        List<Thread> threads = IntStream.range(0, 10)
                .mapToObj(index -> Thread.ofVirtual().unstarted(() -> {
                    System.out.println("BEFORE " + index + " - " + Thread.currentThread());

                    synchronized (lock) {
                        // 시간은 오래 걸리지만 blocking은 없는 사례
                        getFibonacci(40);
                    }
                    
                    System.out.println("AFTER  " + index + " - " + Thread.currentThread());
                }))
                .toList();

        threads.forEach(Thread::start);

        threads.forEach(thread -> {
            try {
                thread.join();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }

    // n번째 피보나치 수
    private static long getFibonacci(int n) {
        if (n <= 1) {
            return n;
        }
        return getFibonacci(n - 1) + getFibonacci(n - 2);
    }

    private static void giveSomeTimeForJFRInit() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}

```

Java 21에서 `java VirtualThreadPinningExample.java` 명령으로 실행하면 다음과 같이 출력된다.

```
BEFORE 9 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-10
BEFORE 1 - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-2
BEFORE 7 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-8
BEFORE 5 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-6
BEFORE 6 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-7
BEFORE 8 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-9
BEFORE 0 - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-1
BEFORE 4 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-5
BEFORE 3 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-4
BEFORE 2 - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-3
AFTER  9 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-10
AFTER  2 - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-3
AFTER  3 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-1
AFTER  4 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-7
AFTER  0 - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-5
AFTER  8 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-8
AFTER  6 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-2
AFTER  5 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-4
AFTER  7 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-6
AFTER  1 - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-9
```

먼저 출력 내용이 무엇을 뜻하는지 알아보자.

예를 들어, `VirtualThread[#37]/runnable@ForkJoinPool-1-worker-10`의 각 부분이 의미하는 바는 다음과 같다.

- `#37`는 가상 스레드 식별자이고,
- `runnable`은 가상 스레드의 상태
- `ForkJoinPool-1`는 가상 스레드를 캐리어 스레드에 마운트하는 스케줄러
- `worker-10`은 가상 스레드가 마운트 된 캐리어 스레드

`VirtualThread[#37]/runnable@ForkJoinPool-1-worker-10`은  
`37번 가상 스레드가 ForkJoinPool-1 스케줄러에 의해 worker-10 캐리어 스레드에 마운트 되어 실행 중인 상태`를 나타낸다.

위 결과를 다시 살펴보면 가장 처음 실행된 `i == 9`일 때를 제외하면 가상 스레드별 BEFORE/AFTER의 캐리어 스레드가 변경된 것을 확인할 수 있다.

즉, 가상 스레드가 `synchronized` 블록을 만나고, 다른 스레드가 이미 락을 가지고 있어 기다릴 때는 캐리어 스레드로부터 언마운트 된다.

결국 다음과 같이 결론 지을 수 있다.

:::info
**가상 스레드는 `synchronized`의 락 획득을 기다리는 동안에는 캐리어 스레드로부터 언마운트 되어 자원 낭비를 유발하지 않는다.**
:::

한편, 가상 스레드와 `synchronized` 얘기가 나오면 꼭 함께 따라나오는 용어가 있다.  
바로 **가상 스레드 고정(pinning)** 이다.

## 고정 발생 시점

JEP 444의 [Executing virtual threads](https://openjdk.org/jeps/444#Executing-virtual-threads) 섹션 중간 정도에 보면 다음과 같이 나와있다.

>There are two scenarios in which a virtual thread cannot be unmounted during blocking operations because it is pinned to its carrier:
>
>    1. When it executes code inside a synchronized block or method, or
>    2. When it executes a native method or a foreign function.

native 메서드나 foreign 함수는 잠시 제쳐두고 `synchronized` 부분만 살펴보면,  
**`synchronized` 블록이나 메서드 내부의 코드를 실행할 때**라고 설명하고 있다.

따라서 다음과 같이 정리할 수 있다.

:::info
**가상 스레드는 `sychronized` 블록이나 메서드에 진입할 때 또는 진입한 직후에 고정 된다.**
:::


## 가상 스레드 고정 문제 진단

앞서 살펴본 JEP 444의 pin 관련 내용 바로 다음에 다음과 같은 문장이 나온다.

>Pinning does not make an application incorrect, but it might hinder its scalability.
>
>고정이 발생한다고 해서 애플리케이션이 잘못 동작하고 있다는 뜻은 아니다. 하지만 고정이 발생하면 확장성을 저해할 수도 있다.

즉, 고정 발생 자체가 문제는 아니다. 그래서 JDK에서도 **가상 스레드 고정 자체를 검출하는 도구가 아니라 가상 스레드 고정 때문에 발생하는 문제를 검출하는 진단 도구를 제공한다.**

### `-Djdk.tracePinnedThreads=full`

다음과 같이 `-Djdk.tracePinnedThreads=full`를 사용해서 예제를 실행해보자.

```
java -Djdk.tracePinnedThreads=full VirtualThreadPinningExample.java
```

출력 결과에 별다른 변화가 없을 것이다.

`synchronized` 블록에 진입하면서 고정이 발생하지만, 그로 인한 문제가 없기 때문에 별다른 변화가 없는 것이다.

이제 고정에 의한 문제가 발생하도록 예제 소스를 수정해서 다시 실행 해보자.

```java
                    synchronized (lock) {
                        // 시간은 오래 걸리지만 blocking은 없는 사례
                        // getFibonacci(40);
                        try {
                            Thread.sleep(50);
                        } catch (InterruptedException e) {
                            throw new RuntimeException(e);
                        }
                    }
```

피보나치 메서드 대신에 블로킹을 유발하는 `Thread.sleep(50)`을 호출한다.  
이렇게 되면 `synchronized` 블록에 들어오면서 가상 스레드가 고정되고,  
그 이후 `Thread.sleep(50)`을 호출하면서 블로킹 되어도 언마운트 되지 못하고 캐리어 스레드를 점유하면서 자원을 낭비하는 문제가 발생한다.

다시 `-Djdk.tracePinnedThreads=full` 플래그를 지정하고 실행하면 다음과 같이 눈에 띄는 변화가 나타난다.

```
java -Djdk.tracePinnedThreads=full VirtualThreadPinningExample.java


BEFORE 7 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-8
BEFORE 0 - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-1
BEFORE 5 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-6
BEFORE 2 - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-3
BEFORE 4 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-5
BEFORE 1 - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-2
BEFORE 3 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-4
BEFORE 6 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-7
BEFORE 9 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-10
BEFORE 8 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-9
VirtualThread[#35]/runnable@ForkJoinPool-1-worker-8 reason:MONITOR
    java.base/java.lang.VirtualThread$VThreadContinuation.onPinned(VirtualThread.java:199)
    java.base/jdk.internal.vm.Continuation.onPinned0(Continuation.java:393)
    java.base/java.lang.VirtualThread.parkNanos(VirtualThread.java:635)
    java.base/java.lang.VirtualThread.sleepNanos(VirtualThread.java:807)
    java.base/java.lang.Thread.sleep(Thread.java:507)
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(VirtualThreadPinningExample.java:22) <== monitors:1
    java.base/java.lang.VirtualThread.run(VirtualThread.java:329)
AFTER  7 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-8
AFTER  8 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-6
AFTER  9 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-5
AFTER  6 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-4
AFTER  3 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-3
AFTER  1 - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-10
AFTER  4 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-7
AFTER  2 - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-2
AFTER  5 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-9
AFTER  0 - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-1
```

이번에는 다음 내용이 추가로 출력되었다.

```
VirtualThread[#35]/runnable@ForkJoinPool-1-worker-8 reason:MONITOR
    java.base/java.lang.VirtualThread$VThreadContinuation.onPinned(VirtualThread.java:199)
    java.base/jdk.internal.vm.Continuation.onPinned0(Continuation.java:393)
    java.base/java.lang.VirtualThread.parkNanos(VirtualThread.java:635)
    java.base/java.lang.VirtualThread.sleepNanos(VirtualThread.java:807)
    java.base/java.lang.Thread.sleep(Thread.java:507)
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(VirtualThreadPinningExample.java:22) <== monitors:1
    java.base/java.lang.VirtualThread.run(VirtualThread.java:329)
```

35번 가상 스레드가 `Thread.sleep()`을 호출하면서 `parkNanos()`를 호출하고 언마운트를 시도하지만,  
`synchronized` 블록의 락(MONITOR)를 가지고 `synchronized` 블록에 진입한 상태, 즉 이미 고정돼 있으므로 언마운트 될 수 없는 상황을 보여주고 있다.

그런데 이상한 점이 하나 있다. 

고정은 10개의 스레드 모두에서 발생하므로 위 로그도 열 번 출력돼야 할 것 같은데 왜 한 번만 출력되는 걸까?  
혹시 고정이 한 번만 발생한 것은 아닐까?

### JFR(JDK Flight Recorder)

JEP 444에서 [가상 스레드 관련해서 네 가지 JFR 이벤트가 추가](https://openjdk.org/jeps/444#JDK-Flight-Recorder-JFR)되었다.

- VirtualThreadStart
- VirtualThreadEnd
- VirtualThreadPinned
- VirtualThreadSubmitFailed

이 중에서 현재 관심사인 VirtualThreadPinned에 대한 설명만 살펴보자.

>jdk.VirtualThreadPinned indicates that a virtual thread was parked while pinned, i.e., without releasing its platform thread (see above). This event is enabled by default, with a threshold of 20ms.
>
>jdk.VirtualThreadPinned는 가상 스레드가 고정된 상태에서 park 되었음을, 즉 가상 스레드가 플랫폼 스레드를 놓아주지 못하고 점유하고 있음을 나타낸다. 이 이벤트는 기본으로 활성화 되며 임계값은 20ms이다.

즉, **원래 park 되면 플랫폼 스레드(캐리어 스레드)를 놓아줘야 하지만 고정돼 있기 때문에 놓아주지 못하고 20ms 이상 점유하면 jdk.VirtualThreadPinned 이벤트가 발생한다**는 얘기다.

이제 JFR을 사용해서 가상 스레드 고정으로 인한 문제를 진단해보자.

다음 명령으로 `recording-sync-sleep-21.jfr` 파일에 가상 스레드 관련 기록을 남길 수 있다.

```
java -Djdk.tracePinnedThreads=full -XX:StartFlightRecording=filename=recording-sync-sleep-21.jfr VirtualThreadPinningExample.java
```

실행 후 다음 명령으로 jfr 파일에서 jdk.VirtualThreadPinned 이벤트 내용을 출력할 수 있다.

```
jfr print --events jdk.VirtualThreadPinned recording-sync-sleep-21.jfr


jdk.VirtualThreadPinned {
  startTime = 21:06:19.179 (2026-02-16)
  duration = 39.2 ms
  eventThread = "" (javaThreadId = 41, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.218 (2026-02-16)
  duration = 54.7 ms
  eventThread = "" (javaThreadId = 34, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.273 (2026-02-16)
  duration = 50.7 ms
  eventThread = "" (javaThreadId = 32, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.324 (2026-02-16)
  duration = 58.6 ms
  eventThread = "" (javaThreadId = 33, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.384 (2026-02-16)
  duration = 54.7 ms
  eventThread = "" (javaThreadId = 39, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.439 (2026-02-16)
  duration = 53.4 ms
  eventThread = "" (javaThreadId = 37, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.493 (2026-02-16)
  duration = 51.0 ms
  eventThread = "" (javaThreadId = 40, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.545 (2026-02-16)
  duration = 54.5 ms
  eventThread = "" (javaThreadId = 38, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.600 (2026-02-16)
  duration = 54.1 ms
  eventThread = "" (javaThreadId = 36, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}

jdk.VirtualThreadPinned {
  startTime = 21:06:19.655 (2026-02-16)
  duration = 51.2 ms
  eventThread = "" (javaThreadId = 35, virtual)
  stackTrace = [
    java.lang.VirtualThread.parkOnCarrierThread(boolean, long) line: 689
    java.lang.VirtualThread.parkNanos(long) line: 648
    java.lang.VirtualThread.sleepNanos(long) line: 807
    java.lang.Thread.sleep(long) line: 507
    ca.bazlur.modern.concurrency.c02.VirtualThreadPinningExample.lambda$main$0(int) line: 22
    ...
  ]
}
```

`jdk.VirtualThreadPinned` 이벤트가 정확히 10번 표시된다. 따라서 고정은 1번이 아니라 10번 발생했다는 사실을 알 수 있다.  

#### `jdk.VirtualThreadPinned`의 진정한 의미

하지만, `jdk.VirtualThreadPinned` 이벤트가 정확히 10번 표시되는 것을 **'고정이 10번 발생했다'라고 해석하는 것은 정확하지 않다**는 점에 유의하자.

고정이 10번 발생한 것은 맞지만, 앞서 `jdk.VirtualThreadPinned`의 설명에서 살펴본 것처럼 **고정으로 인해 캐리어 스레드를 비효율적으로 점유하는 문제가 10번 발생했다**라고 해석해야 정확하다.

그렇다면 고정이 되어도 그로 인해 캐리어 스레드를 비효율적으로 점유하는 문제가 발생하지 않으면 `jdk.VirtualThreadPinned` 이벤트는 발생하지 않는다는 얘긴가?

이를 확인하기 위해 다시 `Thread.sleep()` 대신에, 캐리어 스레드를 비효율적으로 점유하지 않는 피보나치 수 계산으로 되돌려서 JFR 이벤트가 어떻게 기록되는지 살펴보자.

```java
                    synchronized (lock) {
                        // 시간은 오래 걸리지만 blocking은 없는 사례
                        getFibonacci(40);
                        //try {
                        //    Thread.sleep(50);
                        //} catch (InterruptedException e) {
                        //    throw new RuntimeException(e);
                        //}
                    }
```

이번엔 `recording-sync-fibo-21.jfr` 파일에 기록한다.

```
java -Djdk.tracePinnedThreads=full -XX:StartFlightRecording=filename=recording-sync-fibo-21.jfr VirtualThreadPinningExample.java

[0.270s][info][jfr,startup] Started recording 1. No limit specified, using maxsize=250MB as default.
[0.270s][info][jfr,startup] 
[0.270s][info][jfr,startup] Use jcmd 58808 JFR.dump name=1 to copy recording data to file.
BEFORE 2 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-3
BEFORE 1 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-2
BEFORE 8 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-9
BEFORE 4 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-5
BEFORE 0 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-1
BEFORE 9 - VirtualThread[#41]/runnable@ForkJoinPool-1-worker-10
BEFORE 3 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-4
BEFORE 5 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-6
BEFORE 6 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-7
BEFORE 7 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-8
AFTER  2 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-3
AFTER  7 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-9
AFTER  6 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-8
AFTER  5 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-5
AFTER  3 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-1
AFTER  9 - VirtualThread[#41]/runnable@ForkJoinPool-1-worker-7
AFTER  0 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-10
AFTER  4 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-6
AFTER  8 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-4
AFTER  1 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-2

```

jfr 명령으로 출력해보면,

```
jfr print --events jdk.VirtualThreadPinned recording-sync-fibo-21.jfr

```

아무것도 출력되지 않는다.

즉, `synchronized`의 락을 획득하고 블록에 진입하면서 고정은 됐지만,  
그 이후 `synchronized` 블록 내 코드에서 캐리어 스레드를 비효율적으로 점유하며 자원을 낭비하는 문제는 발생하지 않기 때문에, 아무런 `jdk.VirtualThreadPinned` 이벤트도 기록되지 않았다.

따라서 `jdk.VirtualThreadPinned`에 대해 다음과 같이 결론 지을 수 있다.

:::info
- `jdk.VirtualThreadPinned`는 **가상 스레드가 고정될 때 무조건 발생하는 이벤트가 아니라, 고정으로 인해 캐리어 스레드를 비효율적으로 점유하는 문제가 생길 때 발생하는 이벤트다.**
    - 가상 스레드가 고정되더라도 고정 해제 전까지 캐리어 스레드를 비효율적으로 점유하지 않으면 `jdk.VirtualThreadPinned`는 발생하지 않는다.
:::


## 마무리

`synchronized`와 관련한 가상 스레드 이야기는 다음과 같이 정리할 수 있다.

:::info
Java 21에서,
- `synchronized` 블록에 진입하기 전에 락을 획득하기 위해 기다릴 때, 가상 스레드는 캐리어 스레드를 점유하지 않으며 자원을 낭비하지 않는다.
- 고정되는 시점은 `synchronized` 락을 획득해서 블록에 진입할 때 또는 블록에 진입한 직후이다.
  - 고정되는 이유는 `synchronized` 키워드 처리 시 플랫폼 스레드만 락을 획득/보유/반환하는 주체로 작동하도록 되어 있기 때문이다.
- JEP 444에서 추가된 JFR 이벤트인 `jdk.VirtualThreadPinned`는 이름만 보면 고정이 될 때 발생하는 이벤트 같지만,
    - 실제로는 고정으로 인해 캐리어 스레드를 비효율적으로 점유할 때 발생한다.
:::

참고로 글에서는 `synchronized`만 다루었지만, native 메서드에 대해서는 다루지 않았다.

지금까지 알아본 내용은 모두 Java 21 기준이다.

Java 21에서 `synchronized` 관련 자원 낭비는 다음과 같다.

:::warning
- `synchronized` 블록 진입 후에는 가상 스레드가 고정됐기 때문에, 이 때 블로킹 연산을 실행하면 가상 스레드가 언마운트 되지 못하고 캐리어 스레드를 점유한 채 블로킹 연산 완료를 기다리면서 자원을 낭비한다.
:::

`synchronized` 대신에 `ReentrantLock`을 사용하면 고정 현상이 발생하지 않으므로, 블로킹 될 때 가상 스레드가 언마운트 될 수 있으므로 자원 낭비를 피할 수 있다.

직접 작성하는 코드는 `ReentrantLock`을 사용하도록 바꾸면 되지만 사용하는 라이브러리에 있는 `synchronized` 블록은 어쩔 수가 없다.

이런 단점 때문에 Java 21 가상 스레드를 실무에 적용하기엔 현실적으로 무리가 있었다.

하지만 이 문제는 다행스럽게도 [JEP 491](https://openjdk.org/jeps/491)이 적용된 Java 24에서 해결됐다.

JEP 491 내용 중 `synchronized` 관련 내용을 요약하면 다음과 같다.

>- 이전에는 `synchronized` 블록의 락을 획득하고 반환하는 실질적인 주체가 플랫폼 스레드이기 때문에 가상 스레드를 캐리어 스레드에 고정할 수 밖에 없었는데,  
>- JEP 491에서는 가상 스레드도 `synchronized` 블록의 락을 획득/보유/반환할 수 있도록 `synchronized` 처리 로직을 변경해서,  
>    - `synchronized` 블록 진입 전/후 모두에서 가상 스레드가 캐리어 스레드에 고정될 필요없이 언마운트 되어 자원 낭비를 방지할 수 있게 되었다.

## 부록

위 예제에서 `System.out.println("BEFORE " + index + " - " + Thread.currentThread())`와 같이 `println`을 사용하지 않고 `System.out.printf("%d BEFORE - %s%n", index, Thread.currentThread())`와 같이 `printf`를 사용하면 다음과 같이 사뭇 다른 결과가 나온다.

```
0 BEFORE - VirtualThread[#21]/runnable@ForkJoinPool-1-worker-1
1 BEFORE - VirtualThread[#22]/runnable@ForkJoinPool-1-worker-6
2 BEFORE - VirtualThread[#23]/runnable@ForkJoinPool-1-worker-5
3 BEFORE - VirtualThread[#24]/runnable@ForkJoinPool-1-worker-4
4 BEFORE - VirtualThread[#25]/runnable@ForkJoinPool-1-worker-3
5 BEFORE - VirtualThread[#26]/runnable@ForkJoinPool-1-worker-2
6 BEFORE - VirtualThread[#27]/runnable@ForkJoinPool-1-worker-8
8 BEFORE - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-9
9 BEFORE - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-7
7 BEFORE - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-10
0 AFTER  - VirtualThread[#21]/runnable@ForkJoinPool-1-worker-1
7 AFTER  - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-10
9 AFTER  - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-7
8 AFTER  - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-9
6 AFTER  - VirtualThread[#27]/runnable@ForkJoinPool-1-worker-8
5 AFTER  - VirtualThread[#26]/runnable@ForkJoinPool-1-worker-2
4 AFTER  - VirtualThread[#25]/runnable@ForkJoinPool-1-worker-3
3 AFTER  - VirtualThread[#24]/runnable@ForkJoinPool-1-worker-4
2 AFTER  - VirtualThread[#23]/runnable@ForkJoinPool-1-worker-5
1 AFTER  - VirtualThread[#22]/runnable@ForkJoinPool-1-worker-6
```

동일한 `i`값에 대해서는 BEFORE의 worker-no 와 AFTER의 worker-no 가 모두 일치한다.

따라서 위 결과만 보면 가상 스레드가 `synchronized` 블록을 만나서 락 획득을 기다릴 때 캐리어 스레드를 점유하면서 자원을 낭비하는 것처럼 보인다.

왜 이런 일이 발생하는 걸까?

이유는 콘솔에 출력하는 과정 중에도 `synchronized` 블록이 사용되는데,  

>- 출력 전에 formatting이 필요 없는 `println`은 여러 가상 스레드가 `synchronized`(예제에 있는 블록이 아니라 콘솔에 출력하는 과정에 속해 있는 것)에 동시에 도달해서 락을 기다리면서 언마운트 될 가능성이 높고,  
>- 출력 전에 formatting이 필요한 `printf`는 여러 가상 스레드가 각자 formatting 작업을 하느라 시간 차가 발생하고, 콘솔 출력 과정에 속해 있는 `synchronized`에 도달할 때도 시차가 발생하고, 도달 했을 때 이미 앞선 스레드의 출력이 끝나서 락이 반환돼 있는 상태라서 락 획득을 기다릴 필요가 없어서 언마운트 되지 않고 그대로 동일한 스레드에서 계속 실행될 가능성이 높기 때문이다.

결국, **가상 스레드가 `synchronized` 블록의 락 획득을 기다릴 때 캐리어 스레드로부터 언마운트 되어 자원 낭비가 유발되지 않는다**는 사실에는 변함이 없다.

---

## 오류 정정 - 1

이 글의 첫 번째 버전은 '`synchronized` 블록의 락 획득을 기다릴 때 가상 스레드가 언마운트 된다'고 완전히 잘못된 결론을 내렸다. ㅠㅜ

언마운트 된다고 오판한 이유는, 특이하게도 `MessageFormat.format()` 실행만으로도 언마운트가 유발되기 때문이다.

`synchronized` 블록을 아예 없애고 `MessageFormat.format()`를 사용해서 BEFORE/AFTER를 출력하도록 수정해서 테스트해보면 다음과 같이 가상 스레드별 BEFORE/AFTER가 달라지는 것을 확인할 수 있다.

```
java -XX:StartFlightRecording=filename=recording-messageformat-21.jfr,settings=VThreadEvents.jfc VirtualThreadPinningExample.java

[0.270s][info][jfr,startup] Started recording 1. No limit specified, using maxsize=250MB as default.
[0.270s][info][jfr,startup] 
[0.270s][info][jfr,startup] Use jcmd 34551 JFR.dump name=1 to copy recording data to file.
BEFORE 2 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-3
BEFORE 3 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-4
BEFORE 6 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-7
BEFORE 7 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-8
AFTER  6 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-3
AFTER  7 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-10
BEFORE 8 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-9
BEFORE 1 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-2
AFTER  8 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-1
BEFORE 5 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-6
BEFORE 0 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-1
AFTER  5 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-10
AFTER  0 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-3
BEFORE 4 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-5
BEFORE 9 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-10
AFTER  2 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-3
AFTER  3 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-5
AFTER  1 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-5
AFTER  4 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-5
AFTER  9 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-3

```

그래서 새 버전의 글에서는 혼동을 일으키는 `MessageFormat.format()`을 사용하지 않고 `System.out.printf()`를 사용했다.

참고로 JFR 가상 스레드 이벤트 로그를 보면 당연하게도 `VirtualThreadPinned`는 한 번도 발생하지 않는다.

그러면 `MessageFormat.format()`을 사용할 때는 언마운트가 발생하고, `System.out.printf()`를 사용할 때는 언마운트가 발생하지 않는 이유는 무엇일까? 아래는 100% 확실한 것은 아니고 추정이다.

`MessageFormat.format()` 뿐만 아니라 ``"xxxxx".formatted()`` 등 `printf` 가 아닌 다른 방법으로 포맷팅 할 때는 언마운트가 발생한다.  
이런 포맷팅 과정에도 `synchronized` 블록이 여러 곳에서 발견되는데, 그럼에도 불구하고 언마운트가 발생하는 이유는, `synchronized`의 락 획득 대기 중에는 `printf`의 경우와 마찬가지로 언마운트 되지 않는데, 락 획득 대기 외의 여러 처리 과정 어딘가에서 안전하게 언마운트 할 수 있는 지점이 있고, **스케줄링 관점에서 볼 때 언마운트 하는 것이 낫다고 판단해서 그 지점에서 언마운트가 발생**한 것으로 보인다. 그 지점이 정확히 어딘지는 파악하지 못했지만, 논리적으로 설명은 된다.

반대로 `printf` 에서도 `synchronized`의 락 획득 대기 외의 여러 처리 과정이 있을텐데 왜 언마운트 되지 않는 걸까?  
`printf`는 `MessageFormat.format()` 보다 포맷팅 과정이 더 단순하고 짧은 시간에 처리할 수 있어서, **스케줄링 관점에서 언마운트 하지 않고 그냥 동일한 캐리어 스레드에서 계속 수행하는 것이 더 낫다고 판단해서 언마운트가 발생하지 않은 것**으로 보인다.


## 오류 정정 - 2

두 번째 버전에서도 큼지막한 오류가 있었다.

`blocking이 없으면 pinning도 없다`는 완전히 잘못된 해석이다.

`jdk.VirtualThreadPinned`를 이름만 보고 가상 스레드가 고정될 때 발생하는 이벤트라고 오해했기 때문에 발생한 오류였다.

본문에 설명한 것처럼 `jdk.VirtualThreadPinned` 이벤트는 가상 스레드가 고정되어 캐리어 스레드를 비효율적으로 점유하는 문제를 유발할 때 발생한다.

고정은 블로킹이 있든 없든 발생하며, 고정된 상태에서 블로킹이 되고 이로 인해 문제가 될 때에만 `jdk.VirtualThreadPinned` 이벤트가 발생한다.

## 오류 정정 - 3

세 번째 버전에서도 큼지막한 오류가 있었다.

`synchronized block을 만나서 락 획득을 기다릴 때 캐리어 스레드를 점유한다`는 완전히 잘못된 해석이다.

예제 코드에서 아래와 같이 두 개의 문자열을 더해서 출력하고 있는데, 이 부분에서 잘못된 해석을 유발하는 요소가 있었다.

```java
        List<Thread> threads = IntStream.range(0, 10)
                .mapToObj(index -> Thread.ofVirtual().unstarted(() -> {
                    String threadInfo = "BEFORE " + index + " - " + Thread.currentThread() + "\n";  // 여기 (1) !!!

                    synchronized (lock) {
                        // 시간은 오래 걸리지만 blocking은 없는 사례
                        getFibonacci(40);
                    }

                    threadInfo += "AFTER  " + index + " - " + Thread.currentThread() + "\n";  // 여기 (2) !!!
                    System.out.println(threadInfo);  // 여기 (3) !!!
                }))
                .toList();
```

출력 결과를 눈으로 보기 편하도록 `threadInfo`를 하나로 합쳐서 출력을 하게 했는데, 이 부분에서 컴파일러의 코드 최적화가 동작하여 실제 코드와는 다르게 동작을 했다.

(1)에서 사용한 `Thread.currentThread()`의 값이 즉시 평가되어 threadInfo 문자열에 확정될 것이라고 생각했지만,  
실제로는 컴파일러 최적화에 의해 (1)에서 사용한 `Thread.currentThread()`의 값이 즉시 평가되지 않고 (2) 시점에 평가되어,  
(3)에서 출력되는 시점에서는 (1)과 (2)에서의 `Thread.currentThread()` 값이 동일하게 출력됐다.

이는 (1)에서 만들어진 `threadInfo` 문자열이 아무 곳에도 사용되지 않았기 때문에 발생한 현상이다.

다음과 같이 (2) 이전에 `threadInfo` 문자열을 출력에 사용하면 (2) 이전에 `Thread.currentThread()` 값이 평가되어 `threadInfo` 문자열에 확정되며,  
(1)과 (2)에서의 `Thread.currentThread()` 값이 각각 다르게 나온다.

즉, synchronized 블록의 락을 기다릴 때 캐리어 스레드로부터 언마운트 된다.

```java
        List<Thread> threads = IntStream.range(0, 10)
                .mapToObj(index -> Thread.ofVirtual().unstarted(() -> {
                    String threadInfo = "BEFORE " + index + " - " + Thread.currentThread() + "\n";  // 여기 (1) !!!
                    System.out.println(threadInfo);  // 여기 (1-1) !!!

                    synchronized (lock) {
                        // 시간은 오래 걸리지만 blocking은 없는 사례
                        getFibonacci(40);
                    }

                    threadInfo += "AFTER  " + index + " - " + Thread.currentThread() + "\n";  // 여기 (2) !!!
                    System.out.println(threadInfo);  // 여기 (3) !!!
                }))
                .toList();
```
