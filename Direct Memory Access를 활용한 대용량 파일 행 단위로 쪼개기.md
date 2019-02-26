# Direct Memory Access를 활용한 대용량 파일 행 단위로 쪼개기

기가 단위의 파일을 외부에 전송할 일이 생겼다. 

한 방에 보내기엔 너무 커서 파일을 쪼개서(split) 보내려고 하는데, 그마저도 쉽지 않다. 쪼개기 위해 대용량 파일을 읽을 때 이미 수십분 동안 CPU를 너무 잡아 먹어서 이 쪼개는 배치 작업을 스케줄링하는 스케줄러(Quartz)에 문제를 일으킨다.

DMA(Direct Memory Access)를 사용하는 것이 좋겠다.

# DMA

![Imgur](https://i.imgur.com/LNHHp8h.png)

(그림 출처: https://howtodoinjava.com/java/io/how-java-io-works-internally-at-lower-level/)

디스크 컨트롤러는 DMA를 통해 CPU를 건드리지 않고 직접 운영체제 메모리에 접근할 수 있고, 응용 프로그램은 DirectBuffer를 활용해서 JVM 메모리가 아닌 운영체제 메모리에 직접 접근할 수 있다.

장점은 다음과 같다.

1. 디스크에 있는 파일을 운영체제 메모리로 읽어들일 때 CPU를 건드리지 않는다.
1. 운영체제 메모리에 있는 파일 내용을 JVM 내 메모리로 다시 복사할 필요가 없다.
1. JVM 내 힙 메모리를 쓰지 않으므로 GC를 유발하지 않는다.

단점은 다음과 같다.

1. DMA에 사용할 버퍼 생성 시 시간이 더 소요될 수 있다.
1. 바이트 단위로 데이터를 취급하므로, **데이터를 행 단위로 취급하기 불편하다.**

Aㅏ.. 파일 쪼개기 할 때 행 단위로 쪼개야 하는데.. 일단 불편한 것일 뿐 아예 안 되는 것은 아니므로 시도해보자.

종합해보면 **파일의 크기가 대단히 클 때 DMA의 장점을 누릴 수 있고 단점을 피할 수 있다.**

# FileChannel

Java 1.4 부터 도입된 NIO에 `FileChannel`이 포함되어 있는데, `ByteBuffer`를 통해 File I/O를 수행할 수 있다.

대략 다음과 같은 방식으로 사용할 수 있다.

## 파일 읽기 용 FileChannel

```java
FileChannel srcFileChannel = Files.open(Paths.get("/home/homo-efficio/tmp/LargeFile"), StandardOpenOption.READ);
```

## 파일 쓰기 용 FileChannel

```java
FileChannel destFileChannel = Files.open(Paths.get("/home/homo-efficio/tmp/LargeFile"), StandardOpenOption.WRITE);
```

`Path` 말고 `RandomAccessFile`을 활용하는 방법도 있고, 파일 열기 모드에도 `TRUNCATE_EXIST`, `CREATE`, `CREATE_NEW` 등 여러가지가 있고 혼합해서 사용할 수 있다.

# ByteBuffer

DMA를 수행할 수 있는 DirectBuffer를 할당할 수 있는 유일한 Buffer다. DirectBuffer는 다음과 같이 간단하게 생성할 수 있다.

```java
ByteBuffer buffer = ByteBuffer.allocateDirect(100 * 1024 * 1024);
```

버퍼가 direct 인지 아닌지 `buffer.isDirect()`를 통해 판별할 수도 있다.

## 위치 관련 속성

### position

버퍼 내에서 값을 읽거나 쓸 수 있는 **시작 위치**를 나타낸다. 버퍼 내로 1 바이트의 데이터가 추가될 때마다 position 값도 1 증가한다.

`buffer.position()`, `buffer.position(int)`

### limit

버퍼 내에서 값을 읽거나 쓸 수 있는 **끝 위치**를 나타낸다. `limit - 1` 위치까지의 데이터가 읽거나 써진다.

`buffer.position()`, `buffer.position(int)`

### mark

현재 `position` 위치에 표시를 해두고, 나중에 `reset()`을 호출하면 `position`이 표시해둔 위치로 이동한다.

`buffer.mark()`

### capacity

버퍼의 용량(담을 수 있는 데이터의 크기)를 나타낸다.

`buffer.capacity()`


## 위치 관련 메서드

### flip()

`position`을 0으로, `limit`을 읽어들인 데이터의 마지막 바이트+1 위치로 세팅한다. 

버퍼에 있는 데이터를 처음부터 모두 읽을 때 사용한다.

### rewind()

`position`을 0으로, `limit`은 현재 그대로 유지한다.

버퍼에 있는 데이터를 처음부터 현재 `limit` 위치 바로 앞까지 읽을 때 사용한다.

### reset()

`position`을 `mark` 위치로 세팅한다.

### compact()

현재 `position`부터 `limit - 1` 까지의 데이터를 버퍼의 맨 앞으로 복사한 후에, `position`을 복사한 데이터 바로 다음 위치로 이동시키고, `limit`는 `capacity`로 이동한다. 행 단위로 데이터를 다루고자 할 때 핵심 역할을 담당한다. 


## 읽고 쓰는 값 관련 메서드

### get()

`position` 에 있는 값을 읽어서 반환한다.

### put(byte)

`byte`를 `position` 위치에 쓴다.


# 코드

DirectBuffer를 생성하는 것 자체는 그저 전용 메서드 하나 호출하는 것밖에 없어 단순하다. 다만, 줄 바꿈 처리에 좀 신경을 써야 한다.

대용량 파일 대상이므로 행의 크기 보다 버퍼 크기가 크다고 가정했다. 버퍼가 작을 때도 적용되도록 구현할 수는 있겠지만 불필요하게 로직만 복잡해지므로 이 때는 그냥 예외를 던지게 했다.

쉽게 요약하면,

1. 파일에서 버퍼만큼 읽어들인 후,
1. 버퍼에서 LF가 나올 때까지 1 바이트씩 뒤로 이동,
1. LF가 나오면 버퍼의 시작부터 LF까지를 target 파일에 쓰고,
1. LF 뒤에서 버퍼 끝까지의 내용을 `compact()` 메서드를 이용해서 버퍼의 시작 위치로 복사하고,
1. 그 이후의 내용은 다시 1번으로 돌아가서 반복

```java
public static void splitFileIntoDirChannel(String srcFilePath,
                                           String destDirPath,
                                           String splittedFileNameFormat,
                                           ByteBuffer buffer) throws IOException {

    final Path path = Paths.get(srcFilePath);
    final FileChannel srcFileChannel = FileChannel.open(path, StandardOpenOption.READ);
    final byte LINE_FEED = 0x0A;
    final byte CARRIAGE_RETURN = 0x0D;

    int fileCounter = 0;
    int totalReadBytes = 0;
    int totalWriteBytes = 0;
    int readBytes;

    while ((readBytes = srcFileChannel.read(buffer)) >= 0) {
        totalReadBytes += readBytes;

        final int contentLength = buffer.position();
        int newLinePosition = buffer.position();

        if (fileCounter == 0
            && (buffer.get(contentLength - 1) == LINE_FEED
                || buffer.get(contentLength - 1) == CARRIAGE_RETURN)) {
            if (srcFileChannel.isOpen()) srcFileChannel.close();
            throw new IllegalArgumentException(
                    String.format(
                            "Buffer 사이즈가 한 행의 길이보다 커야 합니다. BuffeSize: [%d], LineLength: [%d]",
                            buffer.capacity(), contentLength));
        }

        final FileChannel splittedFileChannel =
                FileChannel.open(Paths.get(destDirPath, String.format(splittedFileNameFormat, ++fileCounter)),
                        StandardOpenOption.TRUNCATE_EXISTING,
                        StandardOpenOption.CREATE,
                        StandardOpenOption.WRITE);

        boolean hasLineFeed = false;
        while (newLinePosition > 0) {
            if (buffer.get(--newLinePosition) == LINE_FEED) {  // 1 byte 씩 뒤로 가면서 줄바꿈 탐색
                buffer.position(0);  // buffer의 처음부터
                buffer.limit(++newLinePosition);  // LINE_FEED 까지 포함해서 write 되도록 limit 조정
                // 버퍼의 [0, limit)의 내용을 splittedFileChannel이 바인딩된 파일에 write
                totalWriteBytes += splittedFileChannel.write(buffer);
                splittedFileChannel.close();
                hasLineFeed = true;
                break;
            }
        }

        if (!hasLineFeed) {
            if (splittedFileChannel.isOpen()) splittedFileChannel.close();
            if (srcFileChannel.isOpen()) srcFileChannel.close();
            throw new IllegalArgumentException("줄바꿈이 하나도 없습니다. 최소 하나 이상 줄바꿈이 있어야 합니다.");
        }

        // compact()를 위해 원래 읽었던 내용의 마지막 바이트 위치+1(==contentLength) 로 limit 설정
        buffer.limit(contentLength);

        // 버퍼의 [position, limit) 의 내용을 [0, limit - position) 으로 복사
        buffer.compact();
        // 복사 후 position은 limit에 위치하며 다음에 파일에서 읽어오는 내용이 position 부터 이어짐

    }

    srcFileChannel.close();

    System.out.println("Total Read  Bytes: " + totalReadBytes);
    System.out.println("Total Write Bytes: " + totalWriteBytes);
}
```


# Benchmark

한 행 당 70바이트 이내, 4,096,000행, 263MB 짜리 csv 파일을 1M, 10M, 50M, 100M 버퍼 크기 단위로 쪼개서 비교해 본 결과는 다음과 같다.

## 수행 시간 비교

버퍼 크기를 1M, 10M, 50M, 100M 를 한 사이클로 해서 2 사이클을 실행해서 수행 시간(초)을 비교해 본 결과,  
전반적으로 DirectBuffer는 일반 Buffer보다 약 10% 정도 빠르고,  
DirectBuffer는 readLine() 방식보다 적게는 약 2배, 많게는 약 5배 정도 빠르다.

버퍼 크기 | 회차 | DirectBuffer | Buffer | readLine
--- | --- | --- | --- | ---
1M | 1 | 1.62 | 1.87 | 3.85
1M | 2 | 1.39 | 1.52 | 3.47
10M | 1 | 0.46 | 0.52 | 2.41
10M | 2 | 0.33 | 0.47 | 2.37
50M | 1 | 1.23 | 2.65 | 2.33
50M | 2 | 1.12 | 1.34 | 2.32
100M | 1 | 1.37 | 1.51 | 2.42
100M | 2 | 1.40 | 1.46 | 2.32

FileChannel과 버퍼를 사용하는 방식은 버퍼 크기가 10M 일 때 다른 크기일 때에 비해 약 3 ~ 4배 정도 빨라지는 것이 특이하다.

## CPU, 메모리 사용량 비교

DirectBuffer, Buffer, readLine() 방식을 순서대로 실행해서 VisualVM으로 측정했다. 그래프에 표시된 것도 맨 왼쪽부터 DirectBuffer, Buffer, readLine() 방식이다.

예상대로 DirectBuffer는 JVM 메모리를 사용하지 않고 운영체제 메모리에 직접 접근하므로 JVM의 힙 메모리 사용량은 거의 없다.  
일반 Buffer는 버퍼 크기에 따라 메모리 사용량도 비례해서 커지며,  
readLine()은 1M, 10M, 50M 단위로 파일을 쪼개도 메모리 사용량은 100M를 넘나든다.

CPU는 DirectBuffer와 Buffer는 아주 큰 차이는 없이 모두 10% 이내다. readLine()에 비해 약 1/3 정도에 불과하지만, readLine() 방식도 30%에 미치지 않은 걸 보면 역시 263MB 짜리 파일은 DirectBuffer 효과를 현저히 누릴 수 있을 정도로 큰 파일은 아니라고 볼 수 있다.

### 1M 버퍼, 1M 단위로 자르기

![Imgur](https://i.imgur.com/6BET1z3.png)

![Imgur](https://i.imgur.com/7dkgJCg.png)

### 10M 버퍼, 10M 단위로 자르기

![Imgur](https://i.imgur.com/wdWRADA.png)

![Imgur](https://i.imgur.com/sdaNr3t.png)

### 50M 버퍼, 50M 단위로 자르기

![Imgur](https://i.imgur.com/G14IWFv.png)

![Imgur](https://i.imgur.com/kEmfUja.png)

### 100M 버퍼, 100M 단위로 자르기

![Imgur](https://i.imgur.com/axhhB1Z.png)

![Imgur](https://i.imgur.com/gJAbo3g.png)


# 정리

>읽을 소스 파일이 기가 단위로 매우 크면 BufferedReader.readLine()은 CPU와 메모리를 엄청 잡아먹는다.
>
>이 때 Java NIO의 FileChannel과 DirectBuffer를 써서 Direct Memory Access를 활용하면 자원 사용량을 줄일 수 있다.
>
>Buffer를 사용하면 자원 사용량과 속도 관점에서는 좋지만 행 단위로 데이터를 처리하는데 불편함이 있다.
>
>Buffer가 제공하는 메서드(position, mark, limit, flip, reset, rewind, compact)를 잘 활용하면 행 단위 처리도 가능하다.

## 읽을거리

- Java I/O를 Low Level에서 아주 잘 설명한 글: https://howtodoinjava.com/java/io/how-java-io-works-internally-at-lower-level/

- Buffer의 사용법을 아주 잘 설명한 글: https://palpit.tistory.com/641

## 기타

DirectBuffer를 사용하더라도 channel의 read/write 메서드가 아니라 buffer의 get/put을 쓰면 DMA가 작동하지 않는다는 글을 봤는데 확인해보진 않았다.

