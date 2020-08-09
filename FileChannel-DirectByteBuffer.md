# FileChannel 과 DirectByteBuffer

Java 4에서 도입된 NIO 덕분에 FileChannel과 ByteBuffer를 이용해서 File I/O 를 수행할 수 있게 됐다.

![Imgur](https://i.imgur.com/vEL6ni9.png)  
그림 출처: https://www.happycoders.eu/java/filechannel-bytebuffer-memory-mapped-file-locks/

NIO의 장점은 https://homoefficio.github.io/2016/08/06/Java-NIO는-생각만큼-non-blocking-하지-않다/ 를 참고하고, 여기에서는 FileChannel과 DirectBuffer 얘기만 다룬다.

ByteBuffer 는 생성되는 위치를 기준으로 크게 나눠보면 JVM Heap 내에 생성되는 HeapByteBuffer와 JVM 밖에 있는 Native 공간에 생성되는 DirectByteBuffer로 나눌 수 있다. 아래 그림에는 먼저 HeapByteBuffer와 MappedByteBuffer로 구분되는 걸로 보이는데 MappedByteBuffer도 Native 공간에 생성되며 파일 일부를 메모리에 매핑한다는 점 외에는 일반적인 direct byte buffer 와 동작이 같다고 API 문서에 나와있다.

![Imgur](https://i.imgur.com/AE4p00B.png)

위 그림에는 안 나와있지만 DirectByteBuffer는 DirectBuffer 인터페이스를 구현하고 있다.

HeapByteBuffer를 사용하면 JVM의 GC에 안전하게 의지할 수 있지만, CPU 개입 없이 I/O를 수행할 수 있고 불필요한 copy 부하가 발생하지 않아 성능적으로 유리한 점이 많은 DMA(DirectMemoryAccess)는 활용할 수 없다.

반대로 DirectByteBuffer를 사용하면 DMA의 혜택을 얻을 수 있지만, JVM의 GC를 벗어나게 되므로 메모리 관리 부담이 생겨난다.

그래서 FileChannel 을 사용할 때 상황에 맞게 HeapByteBuffer나 DirectByteBuffer 중에 골라서 쓰면 될 것 같지만, 실상은 꼭 그렇지는 않다!

이제부터 코드와 함께 그 내부를 살짝 들여다보자.


## FileChannel.write(ByteBuffer)

HeapByteBuffer에 담겨 있는 내용을 파일에 저장하려면 대략 아래와 같은 코드를 사용하게 된다.

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

위 코드에는 DirectByteBuffer가 존재하지 않는다.

그런데 막상 실행해서 [jcmd로 Native 메모리를 모니터링](https://homoefficio.github.io/2020/04/09/Java-Native-Memory-Tracking/) 해보면, Native 메모리를 나타내는 Internal 항목이 위 사용한 Buffer의 크기만큼 증가하는 것을 확인할 수 있다. 대략 다음과 같은 내용이 표시된다.

![](https://i.imgur.com/t9gmDhx.png)

분명히 HeapByteBuffer를 사용했는데 왜 Native 메모리가 동원되는 걸까?


## FileChannelImpl.write(ByteBuffer) 와 그 이후

특별히 지정하지 않는다면 인터페이스인 FileChannel의 구현체로 `sun.nio.ch.FileChannelImpl`이 사용된다. 코드는 아래와 같다.

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

IOUtil.write()는 다음과 같다.

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

오호 특이한 점이 눈에 들어온다. 인자로 받아온 ByteBuffer의 Type이 DirectBuffer이면 writeFromNativeBuffer()를 호출하고 바로 리턴하지만, DirectBuffer가 아니면 `Util.getTemporaryDirectBuffer(rem)` 이렇게 슬그머니 DirectBuffer를 생성한다!! 잡았다 요놈!

Util.getTemporaryDirectBuffer(rem)은 다음과 같다.

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

대충 뭔가 재사용을 위해 캐시 개념을 사용하는 것 같은데 여튼 결국에는 `ByteBuffer.allocateDirect(size)`를 호출한다. `ByteBuffer.allocateDirect(size)`는 다음과 같다.

```java
// java.nio.ByteBuffer

    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
    }
```

즉, 특별한 설정 없이 **일반적인 상황에서 `FileChannel.write()`을 사용하면 항상 DirectByteBuffer가 사용된다.**

좀 당황스럽긴 하지만 뭐 그럴 수도 있기는 한데, 이런 얘기를 왜 API 문서나 기타 자료에서 쉽게 접할 수 없는지는 솔직히 의문이다.

암튼 그래.. 나 몰래 DirectByteBuffer가 내부적으로 사용되네.. 근데 뭐가 문제임? 어쨌거나 잘 돌면 되는 거 아님?

이제부터 현장에서 발생했던 사례 얘기를 풀어본다. 만약 늘 아래 사례와 같이 동작한다면 상당히 심각한 버그라고 볼 수 있는데, 워낙 개발을 못 하는지라 어느 부분에선가 내가 코드를 잘못 짰을 수도 있기 때문에 늘 발생하는 상황이라고 단정할 수는 없다. 어쨌든 호기심이 생긴다면 이어서 쭉 보자.


## DirectByteBuffer 메모리 회수

앞에서 DirectByteBuffer를 사용하면 DMA 혜택을 얻지만 메모리 관리 부담이 생긴다고 했다. DirectByteBuffer 메모리는 어떻게 회수되는 걸까? 여러 자료 찾아봤는데 대략 이런 말로 귀결된다.

>Native 메모리를 참조하는 객체는 결국 JVM Heap 안에 생성되며, 이 객체가 JVM의 GC에 의해 회수되면 이 객체가 참조하는 Native 메모리는 JVM이 아닌 다른 메커니즘에 의해 어쨌든 회수된다.

요는 DirectByteBuffer를 사용해도 간접적이긴 하지만 결국에는 JVM GC에 의해 회수가 시작된다는 얘기다. 오 그럼 다행스럽게도 결국 JVM GC가 챙겨주시는 거네~ 행복~

그런데 어느 정도 시간이 지나면 결국 늘 이 분을 영접하게 되었다.

```
java.lang.OutOfMemoryError: Direct buffer memory
```

처음에는 '아 왜요~~ 저 DirectByteBuffer 안 쓰는데 저한테 왜 이러세요 진짜~' 였다. 그런데 로그를 따라가보니 위에서 설명한 것처럼 나 몰래 응큼하게 내부적으로 DirectByteBuffer가 사용된다는 것까지는 알게 되었다. 그런데 회수는? 나 몰래 만들었으면 나 몰래 해제도 해줘야 하는 거 아님? 난 쪼렙 하수지만 넌 JDK 잖아~

여러 번 테스트 해봤는데 회수 안 해주더라.. JDK고 나발이고 나는 분명히 HeapByteBuffer를 전달해줬는데 나 몰래 DirectByteBuffer랑 바람 피우고, 그것도 모자라 지가 쓴 카드값까지 나한테..

어쨌든 상황을 정리해보면 다음과 같다.

>FileChannel에 데이터를 write할 때는 결국 항상 DirectByteBuffer가 사용되는데,  
>이 DirectByteBuffer가 제대로 회수되지 않는(것 같)다.

자바에 내가 명시적으로 GC를 확실하게 유발할 수 있는 수단이 있는 것도 아닌데.. 망했.. 이러면 FileChannel은 못 쓰는 건데.. API 문서에도 다른 자료에도 왜 시원한 해법이 없지? 설마 아무도 FileChannel을 안 쓰는 건가? 그럴리가.. 내가 잘못 어딘가 짠 거겠지..

별 생각이 다 드는 가운데 여기서 대반전!


## DirectByteBuffer 메모리 회수 방법

DirectByteBuffer는, JVM의 Heap 밖에 있어서 JVM GC가 아닌 다른 메커니즘에 의해 회수된다는 그 **DirectByteBuffer는, 놀랍게도 Java 코드로 명시적으로 바로 GC 할 수 있는 방법이 있었다.** 아무리 검색해도 찾을 수가 없던 귀한 내용이지만 바로 공유한다. 위에서 살펴본 코드 중에 답이 있었다.

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
    private static void free(ByteBuffer buf) {
        ((DirectBuffer)buf).cleaner().clean();
    }
```

응큼하게 바람 피우고 카드값 떠넘기는 HeapByteBuffer와 결별(사실 HeapByteBuffer는 죄가 없다. FileChannel 구현부가 죄인이지)하고 내가 그냥 DirectByteBuffer와 사랑에 빠지기로 했다. 그래서 다음과 같이 HeapByteBuffer를 사용하던 코드를

```java
ByteBuffer buf1 = ByteBuffer.allocate(size);
...
buf1.flip();
fileChannel.write(buf1);
...
ByteBuffer buf2 = ByteBuffer.wrap(byteArray);
fileChannel.write(buf2);
```

다음과 같이 명시적으로 DirectByteBuffer를 생성하고 사용하고 회수하도록 모두 바꾸고나니,

```java
try {
  ByteBuffer directBuffer = ByteBuffer.allocateDirect(size);
  ...
  directBuffer.flip();
  fileChannel.write(directBuffer);
  ...
} finally {
  ((DirectBuffer)directBuffer).cleaner().clean();
}
```

놀랍게도 `((DirectBuffer)directBuffer).cleaner().clean()`가 호출된 후 `jcmd`로 확인해보면 Internal 영역이 DirectByteBuffer 크기만큼 바로 줄어드는 것을 확인할 수 있었다. 그리고 영 반갑지 않은 `java.lang.OutOfMemoryError: Direct buffer memory`도 다시 볼 일 없게 됐다.


## 마무리

>- 특별한 FileChannel 구현체를 사용하지 않는다면, FileChannel에 write 할 때 HeapByteBuffer 를 사용해도 내부적으로 DirectByteBuffer가 사용된다.
>
>- 내부적으로 사용되는 DirectByteBuffer 가 제대로 회수되지 않고 `java.lang.OutOfMemoryError: Direct buffer memory`가 발생할 수 있다.
>
>- 이렇게 FileChannel에 write 하는 코드에 의해 OOM이 발생한다면,
>
>    - HeapByteBuffer 을 사용하지 말고 명시적으로 DirectByteBuffer 를 사용하고,  
>    - `((DirectBuffer)directBuffer).cleaner().clean();` 를 사용해서 명시적으로 해제하자.









