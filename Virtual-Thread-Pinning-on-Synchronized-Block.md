# Virtual Thread Pinning on Synchronized Block

Java 21에서 아래 `synchronized` 블록의 락을 가상 스레드 A가 획득한 상태에서,  
다른 가상 스레드 B가 `synchronized` 블록을 만나면 B는 락을 가지고 있지 않으므로 락 획득 시까지 기다려야 한다.

이 때 가상 스레드 B는 캐리어 스레드로부터 언마운트 될까 아니면 마운트 된 상태로 캐리어 스레드를 점유하며 비효율을 초래할까?

```java
synchronized (lock) {
    try {
        Thread.sleep(50);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
```

## 언마운트 된다

아래의 코드로 확인해 볼 수 있다.

```java
public class ThreadPinningSynchronizedTest {

    private static final Object lock = new Object();

    public static void main(String[] args) {
        List<Thread> threads = IntStream.range(0, 10)
                .mapToObj(index -> Thread.ofVirtual().unstarted(() -> {
                    System.out.println(MessageFormat.format("BEFORE {0} - {1}", index, Thread.currentThread()));

                    synchronized (lock) {
                        try {
                            Thread.sleep(50);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                    }

                    System.out.println(MessageFormat.format("AFTER  {0} - {1}", index, Thread.currentThread()));
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
}

```

Java 21에서 테스트 해본 결과는 다음과 같다. 

```
BEFORE 2 - VirtualThread[#24]/runnable@ForkJoinPool-1-worker-3
BEFORE 7 - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-8
BEFORE 3 - VirtualThread[#25]/runnable@ForkJoinPool-1-worker-4
BEFORE 1 - VirtualThread[#23]/runnable@ForkJoinPool-1-worker-2
BEFORE 5 - VirtualThread[#27]/runnable@ForkJoinPool-1-worker-6
BEFORE 4 - VirtualThread[#26]/runnable@ForkJoinPool-1-worker-5
BEFORE 8 - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-9
BEFORE 0 - VirtualThread[#22]/runnable@ForkJoinPool-1-worker-1
BEFORE 9 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-10
BEFORE 6 - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-7
AFTER  2 - VirtualThread[#24]/runnable@ForkJoinPool-1-worker-3
AFTER  6 - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-7
AFTER  9 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-10
AFTER  0 - VirtualThread[#22]/runnable@ForkJoinPool-1-worker-1
AFTER  4 - VirtualThread[#26]/runnable@ForkJoinPool-1-worker-5
AFTER  8 - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-9
AFTER  5 - VirtualThread[#27]/runnable@ForkJoinPool-1-worker-6
AFTER  1 - VirtualThread[#23]/runnable@ForkJoinPool-1-worker-2
AFTER  3 - VirtualThread[#25]/runnable@ForkJoinPool-1-worker-4
AFTER  7 - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-8
```


`VirtualThread[#24]/runnable@ForkJoinPool-1-worker-3`의 각 부분이 의미하는 바는 다음과 같다.

- `#24`는 가상 스레드 식별자이고,
- `runnable`은 가상 스레드의 상태
- `ForkJoinPool-1`는 가상 스레드를 캐리어 스레드에 마운트하는 스케줄러
- `worker-3`은 가상 스레드가 마운트 된 캐리어 스레드

`VirtualThread[#24]/runnable@ForkJoinPool-1-worker-3`는
**24번 가상 스레드가 ForkJoinPool-1 스케줄러에 의해 worker-3 캐리어 스레드에 마운트 되어 실행 중인 상태**를 나타낸다.

출력 내용을 찬찬히 살펴보면,
처음에 `synchronzied` 블록에 진입한 인덱스 2인 가상 스레드만 BEFORE/AFTER 가 모두 `VirtualThread[#24]/runnable@ForkJoinPool-1-worker-3`로 동일하고,
나머지는 동일한 것도 있고, 아닌 것도 있다.

동일하지 않은 것도 있다는 사실에서 유추할 수 있는 것은 다음과 같다.

> **`synchronized` 블록의 락 획득을 기다리는 동안 가상 스레드는 캐리어 스레드로부터 언마운트 된다.**

'Java 21에서는 가상 스레드가 `synchronized` 블록을 만나면 pinning이 발생한다'라고 피상적으로 알고 있지만,  
엄밀히 말하면 가상 스레드가 `synchronized` 블록에 **진입할 때** pinning이 발생한다.

## 흥미로운 실험

우연히 발견한 재미있는 현상이다.

`System.out.println(MessageFormat.format("BEFORE {0} - {1}", index, Thread.currentThread()));` 이 너무 길어 보여서 짧게 줄인다고 `System.out.printf("BEFORE %d - %s%n", index, Thread.currentThread());`로 변경하고 다시 실행해봤다.

이번에는 다음과 같이 모든 가상 스레드의 BEFORE/AFTER 가 동일하게 나온다.

```
BEFORE 1 - VirtualThread[#23]/runnable@ForkJoinPool-1-worker-2
BEFORE 0 - VirtualThread[#22]/runnable@ForkJoinPool-1-worker-6
BEFORE 2 - VirtualThread[#24]/runnable@ForkJoinPool-1-worker-4
BEFORE 3 - VirtualThread[#25]/runnable@ForkJoinPool-1-worker-7
BEFORE 4 - VirtualThread[#26]/runnable@ForkJoinPool-1-worker-3
BEFORE 6 - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-5
BEFORE 7 - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-1
BEFORE 8 - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-8
BEFORE 9 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-9
BEFORE 5 - VirtualThread[#27]/runnable@ForkJoinPool-1-worker-10
AFTER  1 - VirtualThread[#23]/runnable@ForkJoinPool-1-worker-2
AFTER  5 - VirtualThread[#27]/runnable@ForkJoinPool-1-worker-10
AFTER  9 - VirtualThread[#31]/runnable@ForkJoinPool-1-worker-9
AFTER  8 - VirtualThread[#30]/runnable@ForkJoinPool-1-worker-8
AFTER  7 - VirtualThread[#29]/runnable@ForkJoinPool-1-worker-1
AFTER  6 - VirtualThread[#28]/runnable@ForkJoinPool-1-worker-5
AFTER  4 - VirtualThread[#26]/runnable@ForkJoinPool-1-worker-3
AFTER  3 - VirtualThread[#25]/runnable@ForkJoinPool-1-worker-7
AFTER  2 - VirtualThread[#24]/runnable@ForkJoinPool-1-worker-4
AFTER  0 - VirtualThread[#22]/runnable@ForkJoinPool-1-worker-6
```

출력에 사용하는 메서드만 바꿨을 뿐인데 이렇게나 동작이 달라지는 이유가 뭘까?

문제는 `System.out.printf()`에 있었다.

`System.out.printf()`가 호출하는 `format()`을 실행할 때 다음과 같이 `synchronized` 블록을 실행한다.

```java
    public PrintStream format(String format, Object ... args) {
        try {
            if (lock != null) {
                lock.lock();
                try {
                    implFormat(format, args);
                } finally {
                    lock.unlock();
                }
            } else {
                synchronized (this) {  // 여기!!!
                    implFormat(format, args);
                }
            }
        } catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        } catch (IOException x) {
            trouble = true;
        }
        return this;
    }
```

가상 스레드가 `format()`을 실행하면서 처음 만난 `synchronized` 블록을 실행하면서 캐리어 스레드에 고정됐기 때문에,  
그 이후 예제 코드에서 만난 두 번째 `synchronized` 블록의 락 획득을 기다릴 때는 언마운트 되지 못 하고 캐리어 스레드를 점유한 채로 자원을 낭비하게 된다.

`synchronized` 블록에 의해 발생하는 낭비는 이처럼 쉽게 알아 차리기 어렵다.  
다행스럽게도 [JEP 491](https://openjdk.org/jeps/491)이 적용된 Java 24에서 `synchronized` 블록에 의해 발생하는 낭비 문제가 해결됐다.

JEP 491을 요약하면 다음과 같다.

>- 이전에는 `synchronized` 블록의 락을 획득하고 반환하는 실질적인 주체가 플랫폼 스레드이기 때문에 가상 스레드를 특정 캐리어 스레드에 고정할 수 밖에 없었는데,  
>- JEP 491에서는 가상 스레드도 `synchronized` 블록의 락을 획득하고 반환할 수 있도록 `synchronized` 처리 로직을 변경해서, 가상 스레드가 특정 캐리어 스레드에 고정될 필요가 없도록 개선했다.


