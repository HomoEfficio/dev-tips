# Direct Memory Access를 활용한 대용량 파일 행 단위로 쪼개기

기가 단위의 파일을 외부에 전송할 일이 생겼다. 

한 방에 보내기엔 너무 커서 파일을 쪼개서(split) 보내려고 하는데, 그마저도 쉽지 않다. 쪼개기 위해 대용량 파일을 읽을 때 이미 수십분 동안 CPU를 너무 잡아 먹어서 이 쪼개는 배치 작업을 스케줄링하는 스케줄러(Quartz)에 문제를 일으킨다.

DMA(Direct Memory Access)를 사용하는 것이 좋겠다.

## DMA

![]() DMA 그림

장점은 다음과 같다.

1. 디스크에 있는 파일을 운영체제 메모리로 읽어들일 때 CPU를 건드리지 않는다.
1. 운영체제 메모리에 있는 파일 내용을 JVM 내 메모리로 다시 복사할 필요가 없다.

단점은 다음과 같다.

1. DMA에 사용할 버퍼 할당 시 시간이 더 소요될 수 있다.
1. 바이트 단위로 데이터를 취급하므로, 데이터를 행 단위로 취급하기 불편하다.

Aㅏ.. 파일 쪼개기 할 때 행 단위로 쪼개야 하는데.. 일단 불편한 것일 뿐 아예 안 되는 것은 아니므로 시도해보자.

## FileChannel

Java 1.4 부터 도입된 NIO에 `FileChannel`이 포함되어 있는데, `ByteBuffer`를 통해 File I/O를 수행할 수 있다.

대략 다음과 같은 방식으로 사용할 수 있다.

### 파일 읽기 용 FileChannel

```java
FileChannel srcFileChannel = Files.open(Paths.get("/home/homo-efficio/tmp/LargeFile"), StandardOpenOption.READ);
```

### 파일 쓰기 용 FileChannel

```java
FileChannel destFileChannel = Files.open(Paths.get("/home/homo-efficio/tmp/LargeFile"), StandardOpenOption.WRITE);
```

`Path` 말고 `RandomAccessFile`을 활용하는 방법도 있고, 파일 열기 모드에도 `TRUNCATE_EXIST`, `CREATE`, `CREATE_NEW` 등 여러가지가 있고 혼합해서 사용할 수 있다.

## ByteBuffer

DMA를 수행할 수 있는 DirectBuffer를 할당할 수 있는 유일한 Buffer다. DirectBuffer는 다음과 같이 간단하게 생성할 수 있다.

```java
ByteBuffer buffer = ByteBuffer.allocateDirect(100 * 1024 * 1024);
```

버퍼가 direct 인지 아닌지 `buffer.isDirect()`를 통해 판별할 수도 있다.

### 위치 관련 속성

#### position

버퍼 내에서 값을 읽거나 쓸 수 있는 **시작 위치**를 나타낸다.

`buffer.position()`, `buffer.position(int)`

#### limit

버퍼 내에서 값을 읽거나 쓸 수 있는 **끝 위치**를 나타낸다. `limit - 1` 위치까지의 데이터가 읽거나 써진다.

`buffer.position()`, `buffer.position(int)`

#### mark

현재 `position` 위치에 표시를 해두고, 나중에 reset()을 호출하면 `position`이 이 위치로 이동한다.

`buffer.mark()`

#### capacity

버퍼의 용량(담을 수 있는 데이터의 크기)를 나타낸다.

`buffer.capacity()`


### 위치 관련 메서드

#### flip()

`position`을 0으로, limit을 읽어들인 데이터의 마지막 바이트+1 위치로 세팅한다.

![]()

#### rewind()



#### reset()

`position`을 `mark` 위치로 세팅한다.

#### compact()

현재 `position`부터 `limit - 1` 까지의 데이터를 버퍼의 맨 앞으로 복사한 후에, `position`을 복사한 데이터 바로 다음에 위치시킨다. 행 단위로 데이터를 다루고자 할 때 핵심 역할을 담당한다. limit은 해제?


### 읽고 쓰는 값 관련 메서드

#### get()

`position` 에 있는 값을 읽어서 반환한다.

#### put(byte)

`byte`를 `position` 위치에 쓴다.


## 기타

buffer.put을 쓰면 DMA를 할 수 없으므로 channel.write를 써야만 한다?



