# Java NIO FileChannel 과 DirectByteBuffer

Java 4에서 도입된 NIO 덕분에 `FileChannel`과 `ByteBuffer`를 이용해서 File I/O 를 수행할 수 있게 됐다.

![Imgur](https://i.imgur.com/vEL6ni9.png)  
그림 출처: https://www.happycoders.eu/java/filechannel-bytebuffer-memory-mapped-file-locks/

NIO의 장점은 https://homoefficio.github.io/2016/08/06/Java-NIO는-생각만큼-non-blocking-하지-않다/ 를 참고하고, 여기에서는 `FileChannel`과 `DirectBuffer` 얘기만 다룬다.

`ByteBuffer`는 생성되는 위치를 기준으로 크게 나눠보면 JVM Heap 내에 생성되는 `HeapByteBuffer`와 JVM Heap 밖에 있는 Native 공간에 생성되는 `DirectByteBuffer`로 나눌 수 있다. 아래 그림에는 먼저 `HeapByteBuffer`와 `MappedByteBuffer`로 구분되는 걸로 보이는데 `MappedByteBuffer`도 Native 공간에 생성되며 파일 일부를 메모리에 매핑한다는 점 외에는 일반적인 direct byte buffer 와 동작이 다르지 않다고 [API 문서](https://docs.oracle.com/javase/8/docs/api/java/nio/MappedByteBuffer.html)에 나와있다.

![Imgur](https://i.imgur.com/AE4p00B.png)

위 그림에는 안 나와있지만 `DirectByteBuffer`는 `DirectBuffer` 인터페이스를 구현하고 있다.

`HeapByteBuffer`를 사용하면 JVM의 GC에 안전하게 의지할 수 있지만, CPU 개입 없이 I/O를 수행할 수 있고 불필요한 copy 부하가 발생하지 않아 성능적으로 유리한 점이 많은 DMA(DirectMemoryAccess)는 활용할 수 없다.

반대로 `DirectByteBuffer`를 사용하면 DMA의 혜택을 얻을 수 있지만, JVM의 GC를 벗어나게 되므로 메모리 관리 부담이 생겨난다.

그래서 `FileChannel` 을 사용할 때 상황에 맞게 `HeapByteBuffer`나 `DirectByteBuffer` 중에 골라서 쓰면 될 것 같..지만, 실상은 꼭 그렇지는 않다!

이제부터 코드와 함께 그 내부를 살짝 들여다보자. 앞으로 나오는 내용은 모두 Java 8 기준이다.


# FileChannel.write(ByteBuffer)

`HeapByteBuffer`에 담겨 있는 내용을 파일에 저장하려면 대략 아래와 같은 코드를 사용하게 된다.

```java
String contents = "abcde";
byte[] byteArr = contents.getBytes(StandardCharsets.UTF_8);

ByteBuffer byteBuffer = ByteBuffer.wrap(byteArr);  // HeapByteBuffer를 반환한다.
// 또는
// ByteBuffer byteBuffer = ByteBuffer.allocate(byteArr.length);    // HeapByteBuffer를 반환한다.
// byteBuffer.put(byteArr);
// byteBuffer.flip();

System.out.println("isDirect? " + byteBuffer.isDirect());  // false
fileChannel.write(byteBuffer);
```

위 코드에는 `DirectByteBuffer`가 전혀 나오지 않는다.

그런데 막상 실행해서 [jcmd로 Native 메모리를 모니터링](https://homoefficio.github.io/2020/04/09/Java-Native-Memory-Tracking/) 해보면, `DirectByteBuffer`가 사용하는 Native 메모리를 나타내는 Internal 항목이 위 사용한 Buffer의 크기만큼 증가하는 것을 확인할 수 있다. 대략 다음과 같은 내용이 표시된다.

![](https://i.imgur.com/t9gmDhx.png)

분명히 `HeapByteBuffer`를 사용했는데 왜 Native 메모리가 동원되는 걸까?


# FileChannelImpl.write(ByteBuffer) 와 그 이후

특별히 지정하지 않는다면 인터페이스인 `FileChannel`의 구현체로 `sun.nio.ch.FileChannelImpl`이 사용된다. 코드는 아래와 같다.

```java
// sun.nio.ch.FileChannelImpl

    public int write(ByteBuffer src) throws IOException {
        ensureOpen();
        if (!writable)
            throw new NonWritableChannelException();
        synchronized (positionLock) {
            int n = 0;
            int ti = -1;
            try {
                begin();
                ti = threads.add();
                if (!isOpen())
                    return 0;
                do {
                    n = IOUtil.write(fd, src, -1, nd);  //<=== 여기!!!
                } while ((n == IOStatus.INTERRUPTED) && isOpen());
                return IOStatus.normalize(n);
            } finally {
                threads.remove(ti);
                end(n > 0);
                assert IOStatus.check(n);
            }
        }
    }
```

`IOUtil.write()`는 다음과 같다.

```java
// sun.nio.ch.IOUtil

    static int write(FileDescriptor fd, ByteBuffer src, long position,
                     NativeDispatcher nd)
        throws IOException
    {
        if (src instanceof DirectBuffer)
            return writeFromNativeBuffer(fd, src, position, nd);

        // Substitute a native buffer
        int pos = src.position();
        int lim = src.limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);
        ByteBuffer bb = Util.getTemporaryDirectBuffer(rem);  //<=== 여기!!!
        try {
            bb.put(src);
            bb.flip();
            // Do not update src until we see how many bytes were written
            src.position(pos);

            int n = writeFromNativeBuffer(fd, bb, position, nd);
            if (n > 0) {
                // now update src
                src.position(pos + n);
            }
            return n;
        } finally {
            Util.offerFirstTemporaryDirectBuffer(bb);
        }
    }

```

오호 특이한 점이 눈에 들어온다. 인자로 받아온 `ByteBuffer`의 Type이 `DirectBuffer`이면 `writeFromNativeBuffer()`를 호출하고 반환하지만, `DirectBuffer`가 아니면 `Util.getTemporaryDirectBuffer(rem)` 이렇게 슬그머니 `DirectBuffer`를 생성한다!!

![Imgur](https://i.imgur.com/72E8YHH.jpg?1)

잠시 곁다리로 빠져보자면, 몰래 대체품을 만들어 쓰기는 하지만 그래도 양심은 있는지 다음과 같이 개발자가 `HeapByteBuffer` 생성 시 지정한 크기가 아니라 실제 read/write 할 데이터 크기만큼의 `DirectByteBuffer`만 생성하는 점은 그래도 높이 쳐줄 수 있다. 그런데 이마저도 나중에 살펴볼 `BufferCache` 동작 방식을 생각해보면 좋다고만 할 수는 없다.

```java
int rem = (pos <= lim ? lim - pos : 0);  // 버퍼의 크기가 아니라 실제 read/write 해야할 데이터 크기(limit - position)
ByteBuffer bb = Util.getTemporaryDirectBuffer(rem);  //<=== 여기!!!
```

이어서 계속 따라가보자. `Util.getTemporaryDirectBuffer(rem)`은 다음과 같다.

```java
// sun.nio.ch.Util

    public static ByteBuffer getTemporaryDirectBuffer(int size) {
        // If a buffer of this size is too large for the cache, there
        // should not be a buffer in the cache that is at least as
        // large. So we'll just create a new one. Also, we don't have
        // to remove the buffer from the cache (as this method does
        // below) given that we won't put the new buffer in the cache.
        if (isBufferTooLarge(size)) {
            return ByteBuffer.allocateDirect(size);  //<=== 여기!!!
        }

        BufferCache cache = bufferCache.get();
        ByteBuffer buf = cache.get(size);
        if (buf != null) {
            return buf;
        } else {
            // No suitable buffer in the cache so we need to allocate a new
            // one. To avoid the cache growing then we remove the first
            // buffer from the cache and free it.
            if (!cache.isEmpty()) {
                buf = cache.removeFirst();
                free(buf);
            }
            return ByteBuffer.allocateDirect(size);  //<=== 여기!!!
        }
    }
```

대충 뭔가 재사용을 위해 캐시 개념을 사용하는 것 같은데 여튼 결국에는 `ByteBuffer.allocateDirect(size)`를 호출해서 `DirectByteBuffer`를 생성한다. `ByteBuffer.allocateDirect(size)`는 다음과 같다.

```java
// java.nio.ByteBuffer

    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
    }
```

즉, 특별한 설정 없이 **일반적인 상황에서 `FileChannel.write()`을 사용하면 개발자가 작성한 프로그램 코드에서 `HeapByteBuffer`를 사용했더라도 내부적으로는 그 `HeapByteBuffer`가 사용되지 않고 항상 `DirectByteBuffer`가 사용된다.** 코드 확인 결과 **`FileChannel.read()`도 마찬가지**다.

상당히 당황스럽다. 게다가 이런 얘기를 왜 API 문서나 기타 자료에서 쉽게 접할 수 없는지는 솔직히 의문이다.

암튼 그래.. 내가 쓰라고 한 건 무시하고 나 몰래 `DirectByteBuffer`를 만들어서 사용하네.. 근데 뭐가 문제임? 어쨌거나 잘 돌면 되는 거 아님?

이제부터 현장에서 발생했던 사례 얘기를 풀어본다. 만약 늘 아래 사례와 같이 동작한다면 상당히 심각한 버그라고 볼 수 있는데, 워낙 개발을 못 하는지라 어느 부분에선가 내가 코드를 잘못 짰을 수도 있기 때문에 늘 발생하는 상황이라고 단정할 수는 없다. 어쨌든 호기심이 생긴다면 이어서 쭉 보자.


# DirectByteBuffer 메모리 회수

앞에서 `DirectByteBuffer`를 사용하면 DMA 혜택을 얻지만 메모리 관리 부담이 생긴다고 했다. `DirectByteBuffer` 메모리는 어떻게 회수되는 걸까? 여러 자료 찾아봤는데 대략 이런 말로 귀결된다.

>Native 메모리를 참조하는 객체는 결국 JVM Heap 안에 생성되며,  
>이 객체가 JVM의 GC에 의해 회수되면 이 객체가 참조하는 Native 메모리는  
>JVM이 아닌 다른 메커니즘에 의해 어쨌든 회수된다.

요는 **`DirectByteBuffer`를 사용해도 간접적이긴 하지만 결국에는 JVM GC에 의해 회수가 시작된다**는 얘기다. 오 그럼 바로 회수되는 건 아니지만 다행스럽게도 결국 JVM GC가 챙겨주시는 거나 마찬가지네~

그런데 위 설명과는 다르게 어느 정도 시간이 지나면 결국 늘 이 분을 영접하게 되었다.

```
java.lang.OutOfMemoryError: Direct buffer memory
```

처음에는 '아 왜요~~ 저 `DirectByteBuffer` 쓰지도 않는데 저한테 왜 이러세요 진짜\~' 였다. 그런데 에러 로그를 따라가보니 위에서 설명한 것처럼 나 몰래 응큼하게 내부적으로 `DirectByteBuffer`가 사용된다는 것까지는 알게 되었다. 그런데 메모리 반납은? 나 몰래 만들었으면 나 몰래 반납도 해줘야 하는 거 아님? 난 쪼렙 하수지만 넌 신성한 JDK 잖아!

여러 번 테스트 해봤는데 몇 시간, 심지어 며칠이 지나도 반납 안 해주더라.. JDK고 나발이고 나는 분명히 `HeapByteBuffer`를 전달해줬는데 나 몰래 `DirectByteBuffer`랑 바람 피우고, 그것도 모자라 지가 쓴 카드값까지 나한테..

![Imgur](https://i.imgur.com/Nejwtsv.png)


어쨌든 상황을 정리해보면 다음과 같다.

>일반적으로 `FileChannel`에 데이터를 write할 때는 결국 항상 `DirectByteBuffer`가 사용되는데,  
>`OutOfMemoryError: Direct buffer memory`가 계속 발생하는 걸로 봐서는,  
>`DirectByteBuffer`로 사용된 Native 메모리가 제대로 회수되지 않는(것 같)다.

자바에 내가 명시적으로 GC를 확실하게 유발할 수 있는 수단이 있는 것도 아닌데.. 망했.. 이러면 `FileChannel`은 못 쓰는 건데.. API 문서에도 다른 자료에도 왜 시원한 해법이 없지? 설마 아무도 `FileChannel`을 안 쓰는 건가? 그럴리가.. 내가 어딘가 잘못 짠 거겠지..

별 생각이 다 드는 가운데 여기서 대반전!


# DirectByteBuffer 메모리 회수 방법

`DirectByteBuffer` 메모리는, JVM의 Heap 밖에 있어서 JVM GC가 아닌 다른 메커니즘에 의해 회수된다는 그 **`DirectByteBuffer` 메모리는, 놀랍게도 Java 코드로 명시적으로 바로 회수할 수 있는 방법이 있었다.** 아무리 검색해도 찾을 수가 없던 희귀한 내용이지만 바로 공유한다. 위에서 살펴본 코드 중에 답이 있었다.

```java
// sun.nio.ch.Util

    public static ByteBuffer getTemporaryDirectBuffer(int size) {
        // If a buffer of this size is too large for the cache, there
        // should not be a buffer in the cache that is at least as
        // large. So we'll just create a new one. Also, we don't have
        // to remove the buffer from the cache (as this method does
        // below) given that we won't put the new buffer in the cache.
        if (isBufferTooLarge(size)) {
            return ByteBuffer.allocateDirect(size);
        }

        BufferCache cache = bufferCache.get();
        ByteBuffer buf = cache.get(size);
        if (buf != null) {
            return buf;
        } else {
            // No suitable buffer in the cache so we need to allocate a new
            // one. To avoid the cache growing then we remove the first
            // buffer from the cache and free it.
            if (!cache.isEmpty()) {
                buf = cache.removeFirst();
                free(buf);            //<=== 여기!!!
            }
            return ByteBuffer.allocateDirect(size);
        }
    }
```

엉? `free(buf)`가 있네? 어떻게 생겼나 한 번 볼까? 부왘ㅋㅋ 대박!!

```java
// sun.nio.ch.Util

    private static void free(ByteBuffer buf) {
        ((DirectBuffer)buf).cleaner().clean();
    }
```

응큼하게 바람 피우고 카드값 떠넘기는 `HeapByteBuffer`와 결별(사실 `HeapByteBuffer`는 죄가 없다. `FileChannel` 구현부가 죄인이지)하고 내가 그냥 `DirectByteBuffer`와 사랑에 빠지기로 했다. 그래서 다음과 같이 `HeapByteBuffer`를 사용하던 코드를

```java
ByteBuffer buf1 = ByteBuffer.allocate(size);
...
buf1.flip();
fileChannel.write(buf1);
...
ByteBuffer buf2 = ByteBuffer.wrap(byteArray);
fileChannel.write(buf2);
```

다음과 같이 명시적으로 `DirectByteBuffer`를 생성하고 사용하고 회수하도록 모두 바꾸고나니,

```java
try {
  ByteBuffer directBuffer = ByteBuffer.allocateDirect(size);
  ...
  directBuffer.flip();
  fileChannel.write(directBuffer);
  ...
} catch (Exception e) {
  ...
} finally {
  ((DirectBuffer)directBuffer).cleaner().clean();
}
```

놀랍게도 **`((DirectBuffer)directBuffer).cleaner().clean()`가 호출된 후 `jcmd`로 확인해보면 Internal 영역이 명시적으로 사용한 `DirectByteBuffer` 크기만큼 바로 줄어드는 것을 확인할 수 있었다.** 그리고 영 반갑지 않은 `java.lang.OutOfMemoryError: Direct buffer memory`도 다시 볼 일 없게 됐다.

이렇게 간단하고 직접적인 해결 방법이 이미 존재하는데 왜 그런 게 API 문서에 언급조차 없는지, 그리고 그 해결법도 왜 `public static`이 아니라 `private static` 메서드로 선언해둔 건지는 여전히 의문이다. 이 정도면 거의 일부러 감춰둔 정도 같기도 해서 '이거 써도 되는 거야?'라는 의문조차 들 정도..

그런데 한 가지 궁금한 게 더 있다.

`HeapByteBuffer`를 전달해줘도 내부적으로 응큼하게 `DirectByteBuffer`를 몰래 만드는 `getTemporaryDirectBuffer()` 내부에서, `DirectByteBuffer` 메모리를 회수할 수 있는 `free()` 메서드가 호출되고 있음에도 불구하고 몰래 만들어진 `DirectByteBuffer`가 회수되지 않는 이유는 뭘까? 이건 `BufferCache`를 보면 알 수 있다.


# BufferCache

코드를 다시 보자.

```java
// sun.nio.ch.Util

    public static ByteBuffer getTemporaryDirectBuffer(int size) {
        // If a buffer of this size is too large for the cache, there
        // should not be a buffer in the cache that is at least as
        // large. So we'll just create a new one. Also, we don't have
        // to remove the buffer from the cache (as this method does
        // below) given that we won't put the new buffer in the cache.
        if (isBufferTooLarge(size)) {
            return ByteBuffer.allocateDirect(size);
        }

        BufferCache cache = bufferCache.get();
        ByteBuffer buf = cache.get(size);
        if (buf != null) {
            return buf;
        } else {
            // No suitable buffer in the cache so we need to allocate a new
            // one. To avoid the cache growing then we remove the first
            // buffer from the cache and free it.
            if (!cache.isEmpty()) {
                buf = cache.removeFirst();  //<=== 여기!!!
                free(buf);            //<=== 여기!!!
            }
            return ByteBuffer.allocateDirect(size);
        }
    }
```

`BufferCache`에 동일한 크기의 `DirectByteBuffer`가 있으면 그걸 재사용하고, 없으면 캐시에 있는 다른 크기의 `DirectByteBuffer`를 `free()`를 이용해서 하나 삭제한다. 그렇게 해서 캐쉬의 총 갯수가 늘어나지 않게 한다. 실제로도 이렇게 잘 동작한다.

예를 들어 크기가 10M로 모두 같은 `HeapByteBuffer`를 3개 생성해서 `FileChannel.write()`에 사용하면 내부적으로 `DirectByteBuffer`가 생성되므로 10M 짜리 `DirectByteBuffer` 3개, 총 30M가 사용될 것 같지만, 위에 나오는 BufferCache 덕분에 실제로는 10M 짜리 `DirectByteBuffer` 한 개만 생성되고 재사용된다.

여기까지는 좋은데 문제는 맨 마지막 부분 `return ByteBuffer.allocateDirect(size)`에서 새로 생성된 후 반환되는 `DirectByteBuffer`는 회수되지 않는(걸로 보인)다는 점이다.

`BufferCache`는 아래와 같이 `ThreadLocal`에 담겨서 per-thread로 존재한다. 

```java
// sun.nio.ch.Util

    // Per-thread cache of temporary direct buffers
    private static ThreadLocal<BufferCache> bufferCache =
        new ThreadLocal<BufferCache>()
    {
        @Override
        protected BufferCache initialValue() {
            return new BufferCache();
        }
    };
```

위에서 개발자가 명시적으로 생성한 `HeapByteBuffer` 대신 내부적으로 `DirectByteBuffer`를 생성할 때 실제 read/write 할 만큼의 `DirectByteBuffer`를 생성한다고 했다. 필요한 만큼만 새로 생성하므로 메모리 사용량에 있어서는 유리하지만, 그 필요한 만큼이 그때그때 다른 상황에서는 지금 살펴본 `BufferCache`의 hit율이 떨어져서 `DirectByteBuffer`의 생성 빈도가 많아질 수 있다. [ByteBuffer API](https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html) 문서에 따르면 `DirectByteBuffer`는 메모리 할당/해제 비용이 `HeapByteBuffer`보다 더 크다고 한다. 따라서 `DirectByteBuffer` 생성 빈도가 많으면 성능에 악영향을 미칠 수 있다. 

정리하면, 하나의 스레드에서 이런 비명시적 방식(개발자가 `HeapByteBuffer`를 `FileChannel.write()`에 인자로 전달해줘도 `FileChannelImpl`이 내부적으로 몰래 `DirectByteBuffer`를 생성하는 방식)으로 `DirectByteBuffer`가 생성되면,  
- 크기가 동일한 `HeapByteBuffer`를 여러개 만들어도  
- `BufferCache` 덕분에 해당 스레드 내에서는 `DirectByteBuffer`가 하나만 만들어지고 재사용될 수는 있지만,  
- 그 한 개의 `DirectByteBuffer`가 제대로 회수되지 않으면,
- 새로운 스레드가 실행될 때마다 계속 누적되다가 결국 OutOfMemory 에러를 맞이하게 된다.


# XX:MaxDirectMemorySize 옵션

`DirectByteBuffer`가 사용하는 Native Memory 최대 크기를 지정할 수 있는 옵션이 있다. `XX:MaxDirectMemorySize`인데 `DirectByteBuffer` 메모리가 제대로 회수되지 않고 누적되다가 최대 크기를 넘는다면 어떻게 될까?

1. 오랫동안 사용되지 않고 메모리만 점유하고 있던 `DirectByteBuffer`를 알아서 회수한다.
2. `java.lang.OutOfMemoryError: Direct buffer memory`가 발생한다.

혹시 하는 마음에 1을 기대했는데, 현실은 2다.


# 실제 검증 - 몰래 만들어진 `DirectByteBuffer` 메모리도 회수된다!!

내가 잘못 짰을 수도 있는 로직과 뒤섞인 상태로는, 몰래 만들어지는 `DirectByteBuffer`가 항상 회수되지 않는다고 확언을 할 수 없으므로 실험용 간단한 프로그램을 만들어서 검증해봤다.

https://github.com/HomoEfficio/scratchpad-bytebuffer 를 참고하면 직접 요리조리 해볼 수 있다.

README에 있는대로, 프로그램 실행해서 VisualVM 으로 강제 GC를 시킨 후에 `jcmd`로 확인해보면 Native Memory가 회수되는 것을 확인할 수 있었다.

따라서 **몰래 만들어진 `DirectByteBuffer`도 정상적인 경우라면 해당 `DirectByteBuffer`를 참조하는 객체(A라고 하자)가 GC될 때 `DirectByteBuffer` 메모리도 함께 회수된다.** 라고 결론 지을 수 있겠다.

하지만 그래도 A가 언제 GC 될 지 알 수 없고, GC 전까지 `DirectByteBuffer`는 계속 Native Memory를 점유하게 된다. 아마도 잘못 작성한 코드 때문이겠지만 알 수 없는 이유로 Native Memory 가 장시간 계속 회수되지 않는다면, 우리에겐 `((DirectBuffer)directBuffer).cleaner().clean()` 라는 무기가 있다는 사실을 알아두면 좋다.


# 마무리

>- 특별한 `FileChannel` 구현체를 사용하지 않는다면, `FileChannel`에 read/write 할 때 `HeapByteBuffer`를 사용해도 내부적으로 `DirectByteBuffer`가 사용된다.
>
>- 내부적으로 사용되는 `DirectByteBuffer`가 제대로 회수되지 않으면 `java.lang.OutOfMemoryError: Direct buffer memory`가 발생할 수 있다.
>
>- 이렇게 `FileChannel`에 read/write 하는 코드에 의해 OOM이 발생한다면,
>
>    - `HeapByteBuffer`을 사용하지 말고 명시적으로 `DirectByteBuffer`를 사용하고,  
>    - `((DirectBuffer)directBuffer).cleaner().clean()`를 사용해서 명시적으로 해제하자.


# 함께 보기

- [Java NIO는 생각만큼 non-blocking 하지 않다](https://homoefficio.github.io/2016/08/06/Java-NIO는-생각만큼-non-blocking-하지-않다/)
- [Java NIO Direct Buffer를 이용해서 대용량 파일 행 기준으로 쪼개기](https://homoefficio.github.io/2019/02/27/Java-NIO-Direct-Buffer를-이용해서-대용량-파일-행-기준으로-쪼개기/)
- [대용량 파일을 AsynchronousFileChannel로 다뤄보기](https://homoefficio.github.io/2016/08/13/대용량-파일을-AsynchronousFileChannel로-다뤄보기/)
- [Java Native Memory Tracking](https://homoefficio.github.io/2020/04/09/Java-Native-Memory-Tracking/)



