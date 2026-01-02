# Virtual Thread Pinning on Synchronized Block

Java 21에서 아래 `synchronized` 블록의 락을 가상 스레드 A가 획득한 상태에서,  
다른 가상 스레드 B가 `synchronized` 블록을 만나면 B는 락을 가지고 있지 않으므로 락 획득 시까지 기다려야 한다.

이 때 가상 스레드 B는 캐리어 스레드로부터 언마운트 될까 아니면 마운트 된 상태로 캐리어 스레드를 점유하며 자원을 낭비할까?

```java
synchronized (lock) {
    // 시간은 오래 걸리지만 blocking은 없는 사례
}
```

## 캐리어 스레드를 점유/낭비한다

아래의 코드로 확인해 볼 수 있다.


```java
public class ThreadPinningSynchronizedTest {

    private static final Object lock = new Object();

    public static void main(String[] args) {

        // Give some time for JFR ready
        giveSomeTimeForJFRInit();

        List<Thread> threads = IntStream.range(0, 10)
                .mapToObj(index -> Thread.ofVirtual().unstarted(() -> {
                    System.out.printf("BEFORE %d - %s%n", index, Thread.currentThread());

                    synchronized (lock) {
                        // 시간은 오래 걸리지만 blocking은 없는 사례
                        getFibonacci(40);
                    }

                    System.out.printf("AFTER  %d - %s%n", index, Thread.currentThread());
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

Java 21에서 실행하면 다음과 같이 출력된다.

```
BEFORE 0 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-1
BEFORE 1 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-3
BEFORE 2 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-4
BEFORE 3 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-6
BEFORE 4 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-5
BEFORE 6 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-2
BEFORE 5 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-7
BEFORE 8 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-9
BEFORE 9 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-8
BEFORE 7 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-10
AFTER  0 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-1
AFTER  7 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-10
AFTER  9 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-8
AFTER  8 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-9
AFTER  5 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-7
AFTER  6 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-2
AFTER  4 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-5
AFTER  3 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-6
AFTER  2 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-4
AFTER  1 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-3
```

먼저 출력 내용이 무엇을 뜻하는지 알아보자.

예를 들어, `VirtualThread[#32]/runnable@ForkJoinPool-1-worker-3`의 각 부분이 의미하는 바는 다음과 같다.

- `#32`는 가상 스레드 식별자이고,
- `runnable`은 가상 스레드의 상태
- `ForkJoinPool-1`는 가상 스레드를 캐리어 스레드에 마운트하는 스케줄러
- `worker-3`은 가상 스레드가 마운트 된 캐리어 스레드

`VirtualThread[#32]/runnable@ForkJoinPool-1-worker-3`는  
`32번 가상 스레드가 ForkJoinPool-1 스케줄러에 의해 worker-3 캐리어 스레드에 마운트 되어 실행 중인 상태`를 나타낸다.

위 결과를 다시 살펴보면 가상 스레드별 BEFORE/AFTER 값이 모두 동일한 것을 확인할 수 있다.  
이 말은 곧 가상 스레드가 `synchronized` 블록에 들어가기 전에 마운트 돼 있던 캐리어 스레드와 `synchronized` 블록을 실행하고 빠져나온 후에 마운트 돼 있는 캐리어 스레드가 동일하다는 뜻이다.

따라서 다음과 같이 유추할 수 있다.

```
`synchronized`를 만나면 가상 스레드 고정(pinning) 현상이 생긴다는데,  
역시나 고정되니까 `synchronized` 블록 진입 전과 후의 가상스레드/캐리어스레드 정보가 동일하게 유지되는구나.
```

과연 그럴까?


## blocking이 없으면 pinning도 없다

BEFORE/AFTER 가 동일하게 나오는 이유가 정말로 고정 현상 때문인지 확인해보자.

JFR(JDK Flight Recorder)를 사용하면 [가상 스레드의 네 가지 이벤트](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html#GUID-F24A6627-4599-4FEF-B17F-DF35EBD34F7B)를 발생시키고 추적할 수 있다.

- VirtualThreadStart
- VirtualThreadEnd
- VirtualThreadPinned
- VirtualThreadSubmitFailed

먼저 JFR이 네 가지 이벤트 모두를 발생시키도록 설정하는 VThreadEvents.jfc 파일을 작성한다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration version="2.0" label="Virtual Thread Events"
    description="JFR configuration to record virtual thread events">
  <event name="jdk.VirtualThreadStart">
    <setting name="enabled">true</setting>
  </event>
  <event name="jdk.VirtualThreadEnd">
    <setting name="enabled">true</setting>
  </event>
  <event name="jdk.VirtualThreadPinned">
    <setting name="enabled">true</setting>
  </event>
  <event name="jdk.VirtualThreadSubmitFailed">
    <setting name="enabled">true</setting>
  </event>
</configuration>
```

그리고 다음 명령으로 `VThreadEvents.jfc` 파일을 지정해서 실행하면 `recording-sync-fibo-21.jfr` 파일에 가상 스레드 이벤트 로그가 기록된다.

```
> java -XX:StartFlightRecording=filename=recording-sync-fibo-21.jfr,settings=VThreadEvents.jfc ThreadPinningSynchronizedTest.java
```

실행 결과는 다음과 같다. 앞서 살펴본 것과 마찬가지로 가상 스레드별 BEFORE/AFTER 가 모두 동일하다.

```
[0.283s][info][jfr,startup] Started recording 1. No limit specified, using maxsize=250MB as default.
[0.283s][info][jfr,startup] 
[0.283s][info][jfr,startup] Use jcmd 14851 JFR.dump name=1 to copy recording data to file.
BEFORE 1 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-2
BEFORE 0 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-6
BEFORE 2 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-3
BEFORE 3 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-7
BEFORE 4 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-5
BEFORE 5 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-4
BEFORE 6 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-1
BEFORE 8 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-8
BEFORE 9 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-10
BEFORE 7 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-9
AFTER  1 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-2
AFTER  7 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-9
AFTER  9 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-10
AFTER  8 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-8
AFTER  6 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-1
AFTER  5 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-4
AFTER  4 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-5
AFTER  3 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-7
AFTER  2 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-3
AFTER  0 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-6
```

이제 다음 명령으로 JFR이 기록한 가상 스레드 이벤트 로그를 살펴보자.

```
> jfr print --events jdk.VirtualThreadStart,jdk.VirtualThreadEnd,jdk.VirtualThreadPinned,jdk.VirtualThreadSubmitFailed recording-sync-fibo-21.jfr

jdk.VirtualThreadStart {
  startTime = 14:08:44.206 (2026-01-02)
  javaThreadId = 31
  eventThread = "" (javaThreadId = 31, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.206 (2026-01-02)
  javaThreadId = 32
  eventThread = "" (javaThreadId = 32, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.206 (2026-01-02)
  javaThreadId = 33
  eventThread = "" (javaThreadId = 33, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.206 (2026-01-02)
  javaThreadId = 34
  eventThread = "" (javaThreadId = 34, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.207 (2026-01-02)
  javaThreadId = 35
  eventThread = "" (javaThreadId = 35, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.207 (2026-01-02)
  javaThreadId = 36
  eventThread = "" (javaThreadId = 36, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.207 (2026-01-02)
  javaThreadId = 37
  eventThread = "" (javaThreadId = 37, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.207 (2026-01-02)
  javaThreadId = 39
  eventThread = "" (javaThreadId = 39, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.207 (2026-01-02)
  javaThreadId = 40
  eventThread = "" (javaThreadId = 40, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:08:44.207 (2026-01-02)
  javaThreadId = 38
  eventThread = "" (javaThreadId = 38, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:44.533 (2026-01-02)
  javaThreadId = 32
  eventThread = "" (javaThreadId = 32, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:45.160 (2026-01-02)
  javaThreadId = 38
  eventThread = "" (javaThreadId = 38, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:45.489 (2026-01-02)
  javaThreadId = 40
  eventThread = "" (javaThreadId = 40, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:45.796 (2026-01-02)
  javaThreadId = 39
  eventThread = "" (javaThreadId = 39, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:46.104 (2026-01-02)
  javaThreadId = 37
  eventThread = "" (javaThreadId = 37, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:46.407 (2026-01-02)
  javaThreadId = 36
  eventThread = "" (javaThreadId = 36, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:46.707 (2026-01-02)
  javaThreadId = 35
  eventThread = "" (javaThreadId = 35, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:47.008 (2026-01-02)
  javaThreadId = 34
  eventThread = "" (javaThreadId = 34, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:47.309 (2026-01-02)
  javaThreadId = 33
  eventThread = "" (javaThreadId = 33, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:08:47.607 (2026-01-02)
  javaThreadId = 31
  eventThread = "" (javaThreadId = 31, virtual)
}

```

`VirtualThreadStart`와 `VirtualThreadEnd`만 있을 뿐, **고정 현상에 따라 발생하는 `VirtualThreadPinned`는 하나도 없다.**

즉, **고정 현상이 발생하지 않았음을 의미한다.**

`synchronized` 블록이 분명히 있는데도 왜 고정 현상이 발생하지 않았을까?

이유는 **`synchronized` 블록 내부의 코드에 blocking 연산이 없기 때문이다.**

`synchronized` 블록 내부를 다음과 같이 blocking 연산인 Thread.sleep()을 사용하도록 변경하고, 

```java
                    synchronized (lock) {
                        try {
                            Thread.sleep(50);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                    }
```

다시 JFR 과 함께 실행해서 확인 해보자. 가상 스레드별 BEFORE/AFTER는 아까와 마찬가지로 동일하다.

```
> java -XX:StartFlightRecording=filename=recording-sync-sleep-21.jfr,settings=VThreadEvents.jfc ThreadPinningSynchronizedTest.java

[0.333s][info][jfr,startup] Started recording 1. No limit specified, using maxsize=250MB as default.
[0.333s][info][jfr,startup] 
[0.333s][info][jfr,startup] Use jcmd 23069 JFR.dump name=1 to copy recording data to file.
BEFORE 0 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-1
BEFORE 2 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-9
BEFORE 1 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-2
BEFORE 3 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-7
BEFORE 4 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-5
BEFORE 5 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-8
BEFORE 7 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-6
BEFORE 9 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-10
BEFORE 6 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-4
BEFORE 8 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-3
AFTER  0 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-1
AFTER  8 - VirtualThread[#39]/runnable@ForkJoinPool-1-worker-3
AFTER  6 - VirtualThread[#37]/runnable@ForkJoinPool-1-worker-4
AFTER  9 - VirtualThread[#40]/runnable@ForkJoinPool-1-worker-10
AFTER  7 - VirtualThread[#38]/runnable@ForkJoinPool-1-worker-6
AFTER  5 - VirtualThread[#36]/runnable@ForkJoinPool-1-worker-8
AFTER  4 - VirtualThread[#35]/runnable@ForkJoinPool-1-worker-5
AFTER  3 - VirtualThread[#34]/runnable@ForkJoinPool-1-worker-7
AFTER  1 - VirtualThread[#32]/runnable@ForkJoinPool-1-worker-2
AFTER  2 - VirtualThread[#33]/runnable@ForkJoinPool-1-worker-9
```

이제 가상 스레드 이벤트 로그를 확인 해보자.

```
> jfr print --events jdk.VirtualThreadStart,jdk.VirtualThreadEnd,jdk.VirtualThreadPinned,jdk.VirtualThreadSubmitFailed recording-sync-sleep-21.jfr

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 31
  eventThread = "" (javaThreadId = 31, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 32
  eventThread = "" (javaThreadId = 32, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 33
  eventThread = "" (javaThreadId = 33, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 34
  eventThread = "" (javaThreadId = 34, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 35
  eventThread = "" (javaThreadId = 35, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 36
  eventThread = "" (javaThreadId = 36, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 38
  eventThread = "" (javaThreadId = 38, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 40
  eventThread = "" (javaThreadId = 40, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 37
  eventThread = "" (javaThreadId = 37, virtual)
}

jdk.VirtualThreadStart {
  startTime = 14:31:32.111 (2026-01-02)
  javaThreadId = 39
  eventThread = "" (javaThreadId = 39, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.123 (2026-01-02)
  duration = 53.7 ms
  eventThread = "" (javaThreadId = 31, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.177 (2026-01-02)
  javaThreadId = 31
  eventThread = "" (javaThreadId = 31, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.177 (2026-01-02)
  duration = 51.4 ms
  eventThread = "" (javaThreadId = 39, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.229 (2026-01-02)
  javaThreadId = 39
  eventThread = "" (javaThreadId = 39, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.229 (2026-01-02)
  duration = 55.0 ms
  eventThread = "" (javaThreadId = 37, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.284 (2026-01-02)
  javaThreadId = 37
  eventThread = "" (javaThreadId = 37, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.284 (2026-01-02)
  duration = 55.0 ms
  eventThread = "" (javaThreadId = 40, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.339 (2026-01-02)
  javaThreadId = 40
  eventThread = "" (javaThreadId = 40, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.339 (2026-01-02)
  duration = 50.2 ms
  eventThread = "" (javaThreadId = 38, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.389 (2026-01-02)
  javaThreadId = 38
  eventThread = "" (javaThreadId = 38, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.389 (2026-01-02)
  duration = 50.8 ms
  eventThread = "" (javaThreadId = 36, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.440 (2026-01-02)
  javaThreadId = 36
  eventThread = "" (javaThreadId = 36, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.440 (2026-01-02)
  duration = 54.6 ms
  eventThread = "" (javaThreadId = 35, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.495 (2026-01-02)
  javaThreadId = 35
  eventThread = "" (javaThreadId = 35, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.495 (2026-01-02)
  duration = 54.6 ms
  eventThread = "" (javaThreadId = 34, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.549 (2026-01-02)
  javaThreadId = 34
  eventThread = "" (javaThreadId = 34, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.549 (2026-01-02)
  duration = 53.4 ms
  eventThread = "" (javaThreadId = 32, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.603 (2026-01-02)
  javaThreadId = 32
  eventThread = "" (javaThreadId = 32, virtual)
}

jdk.VirtualThreadPinned {
  startTime = 14:31:32.603 (2026-01-02)
  duration = 54.8 ms
  eventThread = "" (javaThreadId = 33, virtual)
}

jdk.VirtualThreadEnd {
  startTime = 14:31:32.658 (2026-01-02)
  javaThreadId = 33
  eventThread = "" (javaThreadId = 33, virtual)
}

```

`VirtualThreadPinned`가 10번 발생했음을 확인할 수 있다.  

즉, `synchronized` 블록 안에서 블로킹 연산을 실행하면 가상 스레드 고정 현상이 발생하고,  
`synchronized` 블록 안에서 블로킹 연산을 실행하지 않으면 가상 스레드 고정 현상은 발생하지 않는다.

그럼 고정 현상이 발생하지 않았는데도 가상 스레드별 BEFORE/AFTER가 동일하게 나온 이유는 뭘까?


## 언마운트 `하지 않은 것`과 언마운트 `될 수 없는 것`

앞서 살펴본 피보나치 수 구하기, Thread.sleep() 두 가지 케이스에서 모두 가상 스레드별 BEFORE/AFTER가 동일하게 유지됐다.

피보나치 수 구하기에는 블로킹 연산이 없다. 그래서 고정도 발생하지 않는다.  
고정되지 않았는데도 BEFORE/AFTER가 변하지 않는 이유는 언마운트 하지 않는 것이 낫다고 스케줄러가 판단했기 때문이다.

피보나치 수 구하기는 시간은 오래 걸리지만, 그 시간 동안 CPU가 계속 일을 하고 있기 때문에 자원 낭비가 발생하는 것은 아니다. 따라서 자원 낭비를 막는다는 목적으로 언마운트를 할 필요가 없다.  
그래서 이 케이스는 **언마운트를 `하지 않은 것`** 에 해당한다.

Thread.sleep()은 블로킹 연산이다. 그래서 고정도 발생한다.  
고정이 발생했기 때문에 언마운트를 하고 싶어도 할 수가 없다.  
그래서 이 케이스는 **언마운트 `될 수 없는 것`** 에 해당한다.


## 마무리

이로써 다음과 같이 결론 지을 수 있다.

:::info
- Java 21에서,
- `synchronized` 블록에 진입하기 전에 락을 획득하기 위해 기다릴 때, 가상 스레드는 캐리어 스레드를 점유한 상태로 기다리면서 자원을 낭비한다.
    - 가상 스레드 고정 현상이 발생했기 때문에 점유하는 것이 아니고,
    - `synchronized` 키워드 처리 시 플랫폼 스레드만 락을 획득/보유/반환하는 주체로 작동하도록 되어 있기 때문이다.
- `synchronized` 블록이 있다고 해서 항상 가상 스레드 고정 현상이 발생하는 것은 아니다.
    - `synchronized` 관련한 가상 스레드 고정 현상은 `synchronized` 블록 내부에서 발생할 수 있는 현상이다.
    - `synchronized` 블록 내부에서, 블로킹 연산처럼 `synchronized` 내부가 아니었다면 언마운트를 유발했을 코드를 실행할 때 고정 현상이 발생한다.
- 언마운트가 발생하지 않았다고 해서 무조건 고정 현상이 발생했다고 볼 수는 없다.
    - 불필요해서 고정 현상 없이도 **언마운트 하지 않은 것**도 있고, 
    - 필요해도 고정 현상이 발생해서 **언마운트 될 수 없는 것**도 있다.
:::

참고로 글에서는 `synchronized`만 다루었지만, native 메서드 호출 시에도 가상 스레드는 유사한 메커니즘으로 동작한다.

지금까지 알아본 내용은 모두 Java 21 기준이다.

Java 21에서 `synchronized` 관련 자원 낭비는 다음과 같다.

:::warning
- `synchronized` 블록 진입 전에는
    - 가상 스레드가 락 획득을 기다리는 중에도 캐리어 스레드를 점유해서 자원을 낭비하고,
- `synchronized` 블록 진입 후에는 
    - 가상 스레드가 블로킹 연산 실행 시 고정 현상 때문에 언마운트 되지 못하고 캐리어 스레드를 점유한 채 블로킹 연산 완료를 기다리면서 자원을 낭비한다.
:::

이런 단점 때문에 Java 21 가상 스레드를 실무에 적용하기엔 현실적으로 무리가 있었다.

하지만 이 문제는 다행스럽게도 [JEP 491](https://openjdk.org/jeps/491)이 적용된 Java 24에서 해결됐다.

JEP 491 내용 중 `synchronized` 관련 내용을 요약하면 다음과 같다.

>- 이전에는 `synchronized` 블록의 락을 획득하고 반환하는 실질적인 주체가 플랫폼 스레드이기 때문에 가상 스레드를 캐리어 스레드에 고정할 수 밖에 없었는데,  
>- JEP 491에서는 가상 스레드도 `synchronized` 블록의 락을 획득/보유/반환할 수 있도록 `synchronized` 처리 로직을 변경해서,  
>    - `synchronized` 블록 진입 전/후 모두에서 가상 스레드가 캐리어 스레드에 고정될 필요없이 언마운트 되어 자원 낭비를 방지할 수 있게 되었다.



## 오류 정정

이 글의 첫 번째 버전은 '`synchronized` 블록의 락 획득을 기다릴 때 가상 스레드가 언마운트 된다'고 완전히 잘못된 결론을 내렸다. ㅠㅜ

언마운트 된다고 오판한 이유는, 특이하게도 `MessageFormat.format()` 실행만으로도 언마운트가 유발되기 때문이다.

`synchronized` 블록을 아예 없애고 `MessageFormat.format()`를 사용해서 BEFORE/AFTER를 출력하도록 수정해서 테스트해보면 다음과 같이 가상 스레드별 BEFORE/AFTER가 달라지는 것을 확인할 수 있다.

```
java -XX:StartFlightRecording=filename=recording-messageformat-21.jfr,settings=VThreadEvents.jfc ThreadPinningSynchronizedTest.java

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

