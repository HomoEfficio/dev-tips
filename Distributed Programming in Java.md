# Distributed Programming in Java

# Week 1

## Intro to Map-Reduce

![Imgur](https://i.imgur.com/imkZiBC.png)

### map

- 하나의 K,V pair를 취해서 여러 개의 K, V pair 생산
- 병렬로 실행 가능

```
(KA, VA) -> map -> {(KA1, VA1), (KA2, VA2), (KA1, VA2), ...}
(KB, VB) -> map -> {(KB1, VB1), (KB2, VB2), (KB3, VB3), ...}
...
```

### group

- MapReduce 프레임워크에 의해 자동으로 수행
- map 의 결과인 중간 K, V pair들의 K 기준으로 그룹핑

```
{(KA1, {VA1, VA2}), (KA2, {VA2}), ..., (KB1, {VB1}), (KB2, {VB2}), (KB3, {KB3})}

```

### reduce

- grouping 결과인 여러 개의 K, V pair를 K를 기준으로 집계(sum, multiply, ...)

```
{(KA1, VA1 + VA2), (KA2, VA2), ..., (KB1, VB1), (KB2, VB2), (KB3, KB3)}
```

## Hadoop Framework

![Imgur](https://i.imgur.com/RMl6Uqk.png)

- 대규모 병렬 프로그래밍을 쉽게 작성할 수 있게 해주는 프레임워크
- 각 노드는 멀티코어와 스토리지 보유
- Input 데이터에 따라 MR을 위한 자원 배분, 스케줄링
- MR은 함수형 프로그래밍 모델을 따르므로 병렬 연산이 가능해서 분산 수행 가능
- 분산 수행 중 특정 노드가 죽더라도 해당 노드에 배분된 작업만 재 실행하면 복구 가능하므로 Fault Tolerant

## Spark Framework

![Imgur](https://i.imgur.com/Af8MhMK.png)

- 노드에는 멀티코어와 스토리지 외에 메모리도 있으니 느린 스토리지는 적게 쓰고 빠른 메모리를 많이 쓰자

Hadoop | Spark
---- | ----
KV pair | Resilient Distributed Dataset
MR | Transforms(Intermediary - map, filter, join, ...), Actions(Terminal - reduce, collect, ...) 

- Lazy Evaluation
- Caching(Memory): 이 덕분에 성능 대폭 향상. 하지만 디스크보다 비싼 메모리가 충분해야

## TF-IDF Example

![Imgur](https://i.imgur.com/W51sTFg.png)

- Term Frequency(용어 빈도), Inverse Document Frequency(역 문서 빈도)
- 문서간의 유사도 측정에 사용되는 개념
- N개의 문서 D1, D2, ..., DN이 있고 각 문서마다(문서별로 가 아니라) 나오는 용어의 집합을 TERM1, TERM2, ... 라고 하면
  - TFij는 문서 Dj에서 용어 집합의 원소인 용어 Ti이 발견되는 빈도를 의미
- DF1, DF2, ... 는 각 용어 Ti를 포함하는 문서의 갯수를 의미
- 역 문서 빈도 IDFi = N / DFi 는 어떤 용어가 자주 사용되고 어떤 용어가 자주 사용되지 않는지 결정 가능
- 덜 사용되는 용어에게 주는 가중치 Weight(TERMi, Dj) = TFij * log(N / DFi)
- 문서 Dj에서 사용되는 모든 용어 TERMi의 빈도를 추출하는 데 MapReduce 활용 가능
  - ((Dj, TERMi), TFij)
- 1차 MR의 결과를 DFi를 구하는 데 사용 가능

## Page Rank Example

![Imgur](https://i.imgur.com/PKRiboF.png)

- page B의 순위 `RANK(B) = Sum a-SRC(B) (RANK (A) / DEST_COUNT(A))`
  - a-SRC(B)는 B로의 링크를 가지고 있는 모든 페이지의 집합
  - DEST_COUNT(A)는 A에 포함된 모든 외부로의 링크의 수

- page rank 알고리듬을 만들고
- 슈도 코드로 작성한 후
- Spark 문장으로 자연스럽게 작성할 수 있다.
  - flatMapToPair(), reduceByKey(), mapValue()


# Week 2

## Intro to Sockets

![Imgur](https://i.imgur.com/37mX2uy.png)

- 포트에 바인딩해서 먼저 기다리는 것만 다를 뿐 서버 소켓이나 클라이언트 소켓이나 InputStream, OutputStream를 추출해서 R/W를 모두 할 수 있으므로 서버 소켓, 클라이언트 소켓은 연결이 맺어진 후 통신 과정에서는 사실상 기능상의 차이가 없다.

- 서버 쪽: Socket serverSocket = new ServerSocket(3000).accept(), serverSocket.getInputStream(), serverSocket.getOutputStream()
- 클라이언트 쪽: Socket clientSocket = new Socket(serverHost, serverPort), clientSocket.getInputStream(), clientSocket.getOutputStream()

- C/S는 적은 수의 프로세스들 사이에 분산 애플리케이션을 만드는데 주로 사용

## Serialization/Deserialization

![Imgur](https://i.imgur.com/pwft0ee.png)

- IntputStream, OutputStream은 기본적으로 바이트 시퀀스로 통신
- 객체를 바이트 시퀀스로 만들기 위해
  - CUSTOM Ser/De 를 작성
  - XML(metadata overhead가 너무 크다)
  - Java Ser/De
    - implements Serializable
    - transient
  - Interface Definition Language(Protocol Buffer, 다른 언어 플랫폼과 통신 가능, .proto 파일 작성 및 기타 개발 오버헤드)

## Remote Method Invocation(RMI)

![Imgur](https://i.imgur.com/583r6DZ.png)

- Socket과 Ser/De를 바탕으로 나온 고수준 통신 방식
- remote로 호출되거나 반환되는 객체 모두 Serializable 구현 필요
- RMI registry에 등록 필요
- RMI는 초기 셋업이 필요하지만 일단 초기 셋업이 완료되면 실제로는 remote 호출이지만 코드 상에서는 local 호출 하듯이 사용할 수 있음

## Multicast Sockets

![Imgur](https://i.imgur.com/g1niwO0.png)

- Unicast: 1:1
- Broadcast: 1: all in n/w, local network에서만 사용
- Multicast: 1:N, N이 포함된 그룹 선택 가능. 인터넷에서 사용 가능. News feed, Video Conference, Multi-player games
  - DatagramPacket, 64kb
  - 여러개의 Unicast 보다 하나의 Multicast가 효율적

## Pub/Sub Pattern

![Imgur](https://i.imgur.com/DPFNfIj.png)

- Socket 보다 고수준의 통신 패턴
- Publisher, Topic/Message, Consumer
- Multicast와 다르게 Publisher는 Subscriber가 누군지 직접 신경 쓸 필요 없고, Subscriber도 Publisher를 알 필요 없다.
- Publishser와 Subscriber는 토픽만 알면 된다.
- batch 처리, broker 노드를 통해 topic partitioning 가능
- broker 노드는 topic partition을 여러 노드에 복제할 수 있으므로 가용성 증가
- Kafka는 하듭이나 Spark 같은 데이터 분석의 입력 데이터 조달처로 사용되거나, 출력 채널로 자주 사용된다.
  - 따라서 Topic에 들어가는 메시지가 Key/Value 이면 데이터 분석에 유리하다.
  - Key/Value 메시지에는 topic에서 해당 메시지의 위치를 나타내는 인덱스가 포함되기도 한다.

# Week 3

## Single Program Multiple Data(SPMD) Model

![Imgur](https://i.imgur.com/qGE7sOi.png)

- SPMD 추상화
  - 클러스터에 분산되어 있는 노드를 하나의 병렬 컴퓨터처럼 사용할 수 있게 해줌
  - 각 노드는 멀티코어 CPU, 메모리, NIC로 구성되고 클러스터 내 다른 노드와 통신
  - 분산 컴퓨팅에서 가장 어려운 부분은 데이터의 분산
  - 물리적으로는 여러 노드의 개별 메모리에 분산되어 있지만 논리적으로는 하나의 메모리인 것처럼 볼 수 있는 글로벌 뷰 필요
  - 각 노드는 로컬 메모리에 대한 로컬 뷰 보유
  - 프로그래머가 로컬 뷰 <-> 글로벌 뷰 전환 처리를 해야함
  - 각 노드에서 실행되는 프로그램 자체는 동일하지만 데이터는 여러 노드에 분산되어 있는 데이터를 사용하므로 SPMD 모델이라고 부른다.

- MPI(Message Passing Interface)
  - 각 노드에서 하나의 MPI 프로세스를 실행하는 것이 보통이지만 코어 별로 한 개씩의 프로세스 실행도 가능
  - mpi.MPI_Init() 메서드로 프로세스 시작
  - mpi.MPI_Comm_size(mpi.MPI_COMM_WORLD) 메서드로 MPI 애플리케이션에 참여한 프로세스 수 계산
  - MPI_Comm_rank(mpi.MPI_COMM_WORLD) 메서드로 각 프로세스의 rank(0 ~ N-1) 계산  
  - 배열 XG = [0, 1, 2, 3]을 분산 컴퓨터의 메모리에 적재하고 읽는 사례
  - XL.length = XG.length / N
  - XL[i] = XL.length * R + i, R은 해당 프로세스의 순위

- SPMD는 각 프로세스가 서로 다른 프로그램을 실행하는 Client/Server와는 많이 다름

## Point-to-Point Communication(send/recv)

![Imgur](https://i.imgur.com/hJTGToC.png)

- 여러 노드에서 동일 프로그램을 실행한다는 것을 명심
- Rank로 node를 구분
- 보내는 쪽의 메모리 상태와 받는 쪽의 메모리 상태가 다름
- C로 된 MPI 코드: https://en.wikipedia.org/wiki/Message_Passing_Interface#Example_program

## Message Ordering and Deadlock

![Imgur](https://i.imgur.com/JQ0AY0g.png)

### 메시지 순서

- 여러 MPI 프로세스가 각자 보내는 메시지의 순서를 결정할 수 있나?
  - 분산 환경에서는 먼저 보낸 메시지가 먼저 도착한다는 보장이 없다.
  - R0 프로세스에서 R1으로 먼저 보낸 메시지 A가 R2 프로세스가 R3로 늦게 보낸 메시지 B보다 더 늦게 도착한다면, A와 B 중 어느게 더 먼저인가?
- 결정할 기준과 도달 순서가 명확하지 않으므로 메시지 순서는 아래의 값이 같은 메시지 끼리만 전송 시점을 기준으로 순서를 결정할 수 있다.
  - 전송자, 수신자, 데이터 타입, 태그

### 데드락

- 회피 방법
  - 교착 상태를 유발한 프로세스의 한 쪽의 전송/수신 문 순서를 바꾼다.
  - sendrecv(송신 정보, 수신 정보) 메서드를 사용해서 양방향 통신으로 수행한다.

## Nonblocking Communications

![Imgur](https://i.imgur.com/ejoW9zo.png)

- S0, S1, S2, S3의 S는 Statement를 의미
- R0에서 ISend 호출(S1) 후 R1으로부터의 응답이 필요없는 S2는 바로 실행 가능
  - R1으로부터의 응답이 도착해야만 즉 R1으로부터의 응답을 사용하는 S3는 먼저 WAIT()을 호출해서 응답을 받을 때까지 실행되지 않음
- Non-Blocking의 장점은 idle 타임을 줄일 수 있다는 것
- Send/Recv 같은 Blocking 호출은 ISend/IRecv 같은 Non-Blocking 함수 호출후 바로 WAIT()를 호출한 케이스라고 볼 수도 있다.
- Non-Blocking ISend는 Blocking Send와, IRecv는 Send와 짝 지어질 수도 있다.
- Non-Blocking을 사용하면 병렬성을 높일 수 있고 데드락도 방지할 수 있다.

## Collective Communications

![Imgur](https://i.imgur.com/jfKkmVx.png)

- 멀티캐스트와 pub-sub 보다 더 강력한 방식
- R0가 메시지 X를 다른 모든 MPI 프로세스에게 전송
  - R0가 순차적으로 반복을 통해 다른 모든 MPI 프로세스에게 전송
  - 보내야할 대상이 많아지면 R0에 병목

- Collective Communications는 모든 프로세스가 동일한 프로그램을 실행하는 MPI의 특성을 이용
- 모든 MPI 프로세스가 메시지의 소스인 R0의 root를 인자로 해서 MPI_Bcast()를 호출해서 모두에게 메시지 전달
- collective operations에서 유의해야할 점은 모든 프로세스가 동일한 collective operation에 도달할 때까지 기다려야 한다는 점이다. 이런 기다림이 바로 배리어다.
- collective operation이 완료되면 모든 프로세스가 묵시적인 배리어를 통과한다.
- MPI_Bcast()에서는 각 프로세스가 메시지 소스인 root에서 브로드캐스트한 값의 사본을 갖는다.

- MPI는 collective operations 을 폭넓게 지원한다.
- R2가 모든 메시지 Y를 더해야 하는 경우, 더하는 역할을 담당하는 R2의 인덱스를 root로 해서 REDUCE() 오퍼레이션을 호출한다.


B
F
C
A C
A
B E
D
D
A
C
