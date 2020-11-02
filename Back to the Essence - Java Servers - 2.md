# Back to the Essence - Java Servers - 2편

1편에서 블로킹 방식의 싱글 스레드 소켓 서버를 만들어봤고 다음의 문제가 있음을 발견했다.

>- 블로킹 방식의 싱글 스레드 소켓 서버는 시간 끄는 이상한 클라이언트가 하나만 들어와도 서버가 먹통이 되고, 다른 클라이언트까지 먹통될 수 있다.

이제 시간 끄는 이상한 클라이언트가 들어오더라도 서버나 다른 클라이언트가 먹통이 되지 않도록 개선해야 한다.

가장 간단한 방법은 클라이언트의 요청마다 별개의 스레드에서 처리하게 하는 것이다. 가장 널리 사용하는 방식이며 서블릿도 이 방식에 기반을 두고 있고, 서블릿에 기반을 둔 Spring MVC도 이 방식이다.

# Classic IO(BIO) - Multi Thread ServerSocket

서버에서 멀티 스레드를 사용한다면 크게 두 가지 전략이 떠오른다.

1. 멀티 스레드로 `ServerSocket`을 여러 개 띄우고, 이 스레드를 요청 처리 완료때까지 사용한다.
1. `ServerSocket`은 하나의 스레드로 하되, `ServerSocket.accept()`로 받아온 여러 소켓들을 여러 개의 스레드로 처리한다.

일단 1번은 불가다. 아래와 같은 코드는 두 번째 `ServerSocket` 생성 시

```java
ServerSocket serverSocket1 = new ServerSocket(Constants.SERVER_PORT);
ServerSocket serverSocket2 = new ServerSocket(Constants.SERVER_PORT);
```

다음과 같이 `BindException`이 발생한다.

```
Exception in thread "main" java.net.BindException: Address already in use
```

따라서 사용할 수 있는 전략은 2번 뿐이다.

`ServerSocket`을 사용해서 서버 데몬을 만들고, `accept()`로 클라이언트의 연결 요청을 기다리는 것까지는 싱글 스레드 서버와 동일하다. 다만 들어온 연결 요청의 처리를 요청 마다 다른 스레드에서 담당한다는 것만 다르다.

```java
public class EchoSocketServerMultiThread {

    public static void main(String[] args) throws IOException {
        EchoSocketServerMultiThread echoSocketServerMultiThread = new EchoSocketServerMultiThread();
        echoSocketServerMultiThread.start();
    }

    public void start() throws IOException {
        // 50개짜리 스레드 풀
        ExecutorService es = Utils.getCommonExecutorService(50);
        try (ServerSocket serverSocket = new ServerSocket(Constants.SERVER_PORT);
             FileOutputStream fos = Utils.getCommonFileOutputStream()
        ) {
            Utils.serverTimeStamp("===============================", fos);
            Utils.serverTimeStamp("Multi Thread Socket Echo Server 시작", fos);

            while (true) {
                Utils.serverTimeStamp("---------------------------", fos);
                Utils.serverTimeStamp("Echo Server 대기 중", fos);

                // accept() 는 연결 요청이 올 때까지 return 하지 않고 blocking
                Socket acceptedSocket = serverSocket.accept();

                // 연결 요청이 오면 새 thread 에서 요청 처리 로직 수행
                es.execute(() -> {
                    try {
                        Utils.serverTimeStamp("Client 접속!!!", fos);
                        Utils.serverTimeStamp("Echo 시작", fos);
//                    Utils.sleep(500L);
                        EchoProcessor.echo(acceptedSocket);
                        Utils.serverTimeStamp("Echo 완료", fos);
                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }
                });
            }
        }
    }
}


```

`Executors`를 이용해서 단순한 고정 크기 스레드 풀을 만들어 사용한다.

```java
public abstract class Utils {

    // ...

    public static ExecutorService getCommonExecutorService(int nThreads) {
        return Executors.newFixedThreadPool(nThreads);
    }
}
```

# 실습

## 금수저 서버 + 금수저 스레드 풀

50개 짜리 스레드 풀을 사용하면서 1편에서 문제를 유발했던 시나리오 그대로 수행해보자.

1. EchoSocketServerMultiThread 실행
1. EchoSocketClient 실행, EchoSocketClient는 연결 요청 후 5초 후에 메시지를 서버에 전송
1. 5초 이내에 다른 터미널에서 `echo -n '아무거나' | nc localhost 7777` 실행해서 메아리가 터미널에 바로 찍히면 문제 해결

실제로는 실습 편의를 위해 EchoSocketClient가 5초가 아니라 1분 후에 보내도록 설정했다.  
결과는 아래 움짤과 같이 여러 터미널 창에서 동시에 `echo -n '아무거나' | nc localhost 7777`를 실행해도 모두 거의 동시에 메아리가 출력됐다.

![Imgur](https://i.imgur.com/HHYMsq0.gif)

움짤 파일 크기를 작게하기 위해 4개의 터미널창만 캡쳐했지만 실제로는 12개의 창에서 동시에 요청을 날렸다. main 스레드 + EchoSocketClient 요청 처리 스레드 + 12개의 터미널 요청 처리 스레드, 총 14개의 스레드가 사용됐다.

EchoSocketClient 요청은 pool-1-thread-1 스레드가 담당했고, 12개의 터미널 요청은 pool-1-thread-2 ~ pool-1-thread-13 스레드가 각각 처리했다. 로그를 보면 쉽게 알 수 있다. `<==`로 표시한 부분은 설명이다.

```
scratchpad-server git:main 🍺🦑🍺🍕🍺 ❯ tail -f temp.log                                                                                               ✹
[SERVER -            main] 2020-11-02T01:36:54.427095 - ===============================
[SERVER -            main] 2020-11-02T01:36:54.446306 - Multi Thread Socket Echo Server 시작
[SERVER -            main] 2020-11-02T01:36:54.446730 - ---------------------------
[SERVER -            main] 2020-11-02T01:36:54.447015 - Echo Server 대기 중

[SERVER -            main] 2020-11-02T01:37:00.181527 - ---------------------------
[SERVER - pool-1-thread-1] 2020-11-02T01:37:00.181621 - Client 접속!!! <== 1분 후 메시지 보내는 EchoSocketClient 요청 accept
[SERVER -            main] 2020-11-02T01:37:00.181954 - Echo Server 대기 중
[SERVER - pool-1-thread-1] 2020-11-02T01:37:00.181981 - Echo 시작
[CLIENT -            main] 2020-11-02T01:37:00.197423 - Client 시작

[SERVER -            main] 2020-11-02T01:37:08.412302 - ---------------------------
[SERVER - pool-1-thread-2] 2020-11-02T01:37:08.412373 - Client 접속!!! <== 여기서부터는 터미널 nc 클라이언트 요청 accept
[SERVER - pool-1-thread-2] 2020-11-02T01:37:08.412695 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.412671 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T01:37:08.413172 - ---------------------------
[SERVER - pool-1-thread-3] 2020-11-02T01:37:08.413241 - Client 접속!!!
[SERVER - pool-1-thread-3] 2020-11-02T01:37:08.413521 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.413473 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T01:37:08.414077 - ---------------------------
[SERVER - pool-1-thread-4] 2020-11-02T01:37:08.414172 - Client 접속!!!
[SERVER - pool-1-thread-4] 2020-11-02T01:37:08.414427 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.414355 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T01:37:08.414899 - ---------------------------
[SERVER - pool-1-thread-5] 2020-11-02T01:37:08.415059 - Client 접속!!!
[SERVER -            main] 2020-11-02T01:37:08.415181 - Echo Server 대기 중
[SERVER - pool-1-thread-5] 2020-11-02T01:37:08.415326 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.415654 - ---------------------------
[SERVER - pool-1-thread-6] 2020-11-02T01:37:08.415793 - Client 접속!!!
[SERVER -            main] 2020-11-02T01:37:08.415901 - Echo Server 대기 중
[SERVER - pool-1-thread-6] 2020-11-02T01:37:08.416132 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.416741 - ---------------------------
[SERVER - pool-1-thread-7] 2020-11-02T01:37:08.417166 - Client 접속!!!
[SERVER -            main] 2020-11-02T01:37:08.417073 - Echo Server 대기 중
[SERVER - pool-1-thread-7] 2020-11-02T01:37:08.417448 - Echo 시작
[SERVER - pool-1-thread-8] 2020-11-02T01:37:08.417732 - Client 접속!!!
[SERVER - pool-1-thread-8] 2020-11-02T01:37:08.418018 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.417644 - ---------------------------
[SERVER -            main] 2020-11-02T01:37:08.418353 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T01:37:08.418818 - ---------------------------
[SERVER - pool-1-thread-9] 2020-11-02T01:37:08.418987 - Client 접속!!!
[SERVER -            main] 2020-11-02T01:37:08.419078 - Echo Server 대기 중
[SERVER - pool-1-thread-9] 2020-11-02T01:37:08.419201 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.419664 - ---------------------------
[SERVER - pool-1-thread-10] 2020-11-02T01:37:08.419741 - Client 접속!!!
[SERVER -            main] 2020-11-02T01:37:08.419985 - Echo Server 대기 중
[SERVER - pool-1-thread-10] 2020-11-02T01:37:08.419997 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.420454 - ---------------------------
[SERVER - pool-1-thread-11] 2020-11-02T01:37:08.421378 - Client 접속!!!
[SERVER -            main] 2020-11-02T01:37:08.421536 - Echo Server 대기 중
[SERVER - pool-1-thread-11] 2020-11-02T01:37:08.421598 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.421942 - ---------------------------
[SERVER - pool-1-thread-12] 2020-11-02T01:37:08.422346 - Client 접속!!!
[SERVER -            main] 2020-11-02T01:37:08.422542 - Echo Server 대기 중
[SERVER - pool-1-thread-12] 2020-11-02T01:37:08.422558 - Echo 시작
[SERVER -            main] 2020-11-02T01:37:08.423231 - ---------------------------
[SERVER - pool-1-thread-13] 2020-11-02T01:37:08.423331 - Client 접속!!!
[SERVER -            main] 2020-11-02T01:37:08.423519 - Echo Server 대기 중
[SERVER - pool-1-thread-13] 2020-11-02T01:37:08.423555 - Echo 시작
[SERVER - pool-1-thread-12] 2020-11-02T01:37:08.440014 - Echo 완료
[SERVER - pool-1-thread-7] 2020-11-02T01:37:08.440467 - Echo 완료
[SERVER - pool-1-thread-4] 2020-11-02T01:37:08.440014 - Echo 완료
[SERVER - pool-1-thread-3] 2020-11-02T01:37:08.440820 - Echo 완료
[SERVER - pool-1-thread-13] 2020-11-02T01:37:08.441453 - Echo 완료
[SERVER - pool-1-thread-11] 2020-11-02T01:37:08.441606 - Echo 완료
[SERVER - pool-1-thread-9] 2020-11-02T01:37:08.442308 - Echo 완료
[SERVER - pool-1-thread-2] 2020-11-02T01:37:08.442538 - Echo 완료
[SERVER - pool-1-thread-6] 2020-11-02T01:37:08.443030 - Echo 완료
[SERVER - pool-1-thread-10] 2020-11-02T01:37:08.442771 - Echo 완료
[SERVER - pool-1-thread-8] 2020-11-02T01:37:08.443385 - Echo 완료
[SERVER - pool-1-thread-5] 2020-11-02T01:37:08.443648 - Echo 완료
[CLIENT -            main] 2020-11-02T01:38:00.227402 - 메시지 전송 시작 <== EchoSocketClient 가 1분 후 메시지 전송
[CLIENT -            main] 2020-11-02T01:38:00.238871 - 메시지 print 완료
[CLIENT -            main] 2020-11-02T01:38:00.242828 - 메시지 flush 완료
[CLIENT -            main] 2020-11-02T01:38:00.243060 - 서버 Echo 대기...
[CLIENT -            main] 2020-11-02T01:38:00.261370 - 서버 Echo 도착
[SERVER - pool-1-thread-1] 2020-11-02T01:38:00.261379 - Echo 완료 <== pool-1-thread-1은 클라이언트에 의해 1분 동안 블록돼 있는 동안 0.5초 지나가므로 클라이언트로부터 메시지 받자마자 Echo
[CLIENT -            main] 2020-11-02T01:38:00.263653 - 서버 Echo msg: Server Echo - 안녕, echo server
```

## 흑수저 서버 + 흑수저 스레드 풀 

이번에는 메아리 처리에 0.5초가 걸리는 후진 서버에서 스레드도 4개만 사용해서 테스트해보자.

메아리 처리에 0.5초가 걸리도록 EchoProcessor에서 주석 처리 돼 있던 `sleep()`의 주석을 해제한다.

```java
public abstract class EchoProcessor {

    private static final FileOutputStream fos = Utils.getCommonFileOutputStream();

    public static void echo(Socket socket) throws IOException {
        try (BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
             PrintWriter out = new PrintWriter(socket.getOutputStream())
        ) {
            String clientMessage = in.readLine();  // in에 읽을 게 들어올 때까지 blocking
            String serverMessage = "Server Echo - " + clientMessage + System.lineSeparator();
            Utils.sleep(500L);  // Echo 처리에 0.5초가 걸리도록 주석 해제
            out.println(serverMessage);
            out.flush();
        }
    }
}
```

EchoSocketServerMultiThread의 스레드 풀을 스레드 4개만 사용하도록 `ExecutorService es = Utils.getCommonExecutorService(4);`로 바꿔서 흑수저 스레드 풀을 만든다. 그리고 앞에서와 마찬가지로 테스트 해보면 아래 움짤처럼 터미널에 표시되는 메아리에 시차가 조금 발생하는 것을 확인할 수 있다.

![Imgur](https://i.imgur.com/zhsVRRw.gif)

**요청 처리엔 시간이 필요하고, 스레드도 메모리 및 Context Switching 부담이 있어서 마냥 늘릴 수만은 없으므로, 이 흑수저 서버 + 흑수저 스레드 풀 시나리오가 현실에 존재하는 시나리오에 가깝다**고 할 수 있다.

1분 지연 Java Socket Client 요청 1개, 12개의 터미널 nc 요청 처리 로그는 다음과 같다. 중간중간 0.5초 정도 차이나는 부분을 유의해서 살펴보자. 자세한 내용은 `<==`로 설명을 추가했다.

```
scratchpad-server git:main 🍺🦑🍺🍕🍺 ❯ tail -f temp.log        
[SERVER -            main] 2020-11-02T12:29:29.653111 - ===============================
[SERVER -            main] 2020-11-02T12:29:29.675600 - Multi Thread Socket Echo Server 시작
[SERVER -            main] 2020-11-02T12:29:29.676223 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:29.676571 - Echo Server 대기 중

[SERVER -            main] 2020-11-02T12:29:36.000378 - ---------------------------
[SERVER - pool-1-thread-1] 2020-11-02T12:29:36.000475 - Client 접속!!! <== 1분 후 메시지 보내는 EchoSocketClient 요청
[SERVER - pool-1-thread-1] 2020-11-02T12:29:36.000856 - Echo 시작
[SERVER -            main] 2020-11-02T12:29:36.000830 - Echo Server 대기 중
[CLIENT -            main] 2020-11-02T12:29:36.015713 - Client 시작
    
[SERVER -            main] 2020-11-02T12:29:43.072899 - ---------------------------
[SERVER - pool-1-thread-2] 2020-11-02T12:29:43.072977 - Client 접속!!! <== 터미널 요청 1 accept()
[SERVER -            main] 2020-11-02T12:29:43.073253 - Echo Server 대기 중 <== main 스레드는 블록되지 않고 계속 요청을 받아 큐에 넣고 다음 요청 대기
[SERVER - pool-1-thread-2] 2020-11-02T12:29:43.073283 - Echo 시작 <== 요청 1 Echo 시작
[SERVER - pool-1-thread-3] 2020-11-02T12:29:43.073725 - Client 접속!!! <== 터미널 요청 2 accept()
[SERVER -            main] 2020-11-02T12:29:43.073657 - ---------------------------
[SERVER - pool-1-thread-3] 2020-11-02T12:29:43.073938 - Echo 시작
[SERVER -            main] 2020-11-02T12:29:43.073984 - Echo Server 대기 중 <== main 스레드는 블록되지 않고 계속 요청을 받아 큐에 넣고 다음 요청 대기
[SERVER -            main] 2020-11-02T12:29:43.074461 - ---------------------------
[SERVER - pool-1-thread-4] 2020-11-02T12:29:43.074550 - Client 접속!!! <== 터미널 요청 3 accept()
[SERVER -            main] 2020-11-02T12:29:43.074694 - Echo Server 대기 중 <== main 스레드는 블록되지 않고 계속 요청을 받아 큐에 넣고 다음 요청 대기
[SERVER - pool-1-thread-4] 2020-11-02T12:29:43.074716 - Echo 시작
[SERVER -            main] 2020-11-02T12:29:43.075061 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.075294 - Echo Server 대기 중 <== main 스레드는 블록되지 않고 계속 요청을 받아 큐에 넣고 다음 요청 대기
[SERVER -            main] 2020-11-02T12:29:43.075582 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.075844 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T12:29:43.076123 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.076339 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T12:29:43.076611 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.076860 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T12:29:43.080305 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.080641 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T12:29:43.080866 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.081025 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T12:29:43.081308 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.081528 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T12:29:43.081733 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.081907 - Echo Server 대기 중
[SERVER -            main] 2020-11-02T12:29:43.082124 - ---------------------------
[SERVER -            main] 2020-11-02T12:29:43.082250 - Echo Server 대기 중 <== main 스레드는 블록되지 않고 계속 요청을 받아 큐에 넣고 다음 요청 대기
[SERVER - pool-1-thread-2] 2020-11-02T12:29:43.593116 - Echo 완료 <== 요청 1 Echo 완료, 0.5초 소요
[SERVER - pool-1-thread-4] 2020-11-02T12:29:43.593116 - Echo 완료
[SERVER - pool-1-thread-3] 2020-11-02T12:29:43.593413 - Echo 완료
[SERVER - pool-1-thread-2] 2020-11-02T12:29:43.593569 - Client 접속!!! <== 요청 1 처리 완료 후 큐에 있던 후속 요청 A accept()
[SERVER - pool-1-thread-4] 2020-11-02T12:29:43.593575 - Client 접속!!! <== 요청 3 처리 완료 후 큐에 있던 후속 요청 C accept()
[SERVER - pool-1-thread-2] 2020-11-02T12:29:43.593817 - Echo 시작 <== 요청 A
[SERVER - pool-1-thread-3] 2020-11-02T12:29:43.593683 - Client 접속!!! <== 요청 2 처리 완료 후 큐에 있던 후속 요청 B accept()
[SERVER - pool-1-thread-4] 2020-11-02T12:29:43.593889 - Echo 시작
[SERVER - pool-1-thread-3] 2020-11-02T12:29:43.594157 - Echo 시작
[SERVER - pool-1-thread-2] 2020-11-02T12:29:44.097963 - Echo 완료 <== 요청 A
[SERVER - pool-1-thread-4] 2020-11-02T12:29:44.097963 - Echo 완료
[SERVER - pool-1-thread-2] 2020-11-02T12:29:44.098370 - Client 접속!!!
[SERVER - pool-1-thread-4] 2020-11-02T12:29:44.098411 - Client 접속!!!
[SERVER - pool-1-thread-4] 2020-11-02T12:29:44.098672 - Echo 시작
[SERVER - pool-1-thread-2] 2020-11-02T12:29:44.098608 - Echo 시작
[SERVER - pool-1-thread-3] 2020-11-02T12:29:44.097963 - Echo 완료
[SERVER - pool-1-thread-3] 2020-11-02T12:29:44.099086 - Client 접속!!!
[SERVER - pool-1-thread-3] 2020-11-02T12:29:44.099283 - Echo 시작
[SERVER - pool-1-thread-4] 2020-11-02T12:29:44.603912 - Echo 완료
[SERVER - pool-1-thread-2] 2020-11-02T12:29:44.604235 - Echo 완료
[SERVER - pool-1-thread-4] 2020-11-02T12:29:44.604316 - Client 접속!!!
[SERVER - pool-1-thread-4] 2020-11-02T12:29:44.604478 - Echo 시작
[SERVER - pool-1-thread-2] 2020-11-02T12:29:44.604473 - Client 접속!!!
[SERVER - pool-1-thread-3] 2020-11-02T12:29:44.604637 - Echo 완료
[SERVER - pool-1-thread-2] 2020-11-02T12:29:44.604910 - Echo 시작
[SERVER - pool-1-thread-3] 2020-11-02T12:29:44.604962 - Client 접속!!!
[SERVER - pool-1-thread-3] 2020-11-02T12:29:44.605142 - Echo 시작
[SERVER - pool-1-thread-3] 2020-11-02T12:29:45.107464 - Echo 완료
[SERVER - pool-1-thread-4] 2020-11-02T12:29:45.107464 - Echo 완료
[SERVER - pool-1-thread-2] 2020-11-02T12:29:45.107464 - Echo 완료
[CLIENT -            main] 2020-11-02T12:30:36.051155 - 메시지 전송 시작 <== 1분 후 메시지 보내는 Java Socket Client
[CLIENT -            main] 2020-11-02T12:30:36.068386 - 메시지 print 완료
[CLIENT -            main] 2020-11-02T12:30:36.073274 - 메시지 flush 완료 <== 메시지 전송
[CLIENT -            main] 2020-11-02T12:30:36.073655 - 서버 Echo 대기...
[SERVER - pool-1-thread-1] 2020-11-02T12:30:36.581245 - Echo 완료 <== 메시지 전송 받은 후 0.5초 후에 Echo 완료
[CLIENT -            main] 2020-11-02T12:30:36.582077 - 서버 Echo 도착
[CLIENT -            main] 2020-11-02T12:30:36.585161 - 서버 Echo msg: Server Echo - 안녕, echo server
```

# 정리

>- `ServerSocket.accept()`로 클라이언트의 요청을 받아들이는 건 하나의 스레드에서 담당하고,
>- 요청의 처리는 요청마다 별도의 스레드에서 처리하게 하면,
>- 시간을 오래 끄는 클라이언트가 일부 있더라도 서버와 나머지 클라이언트는 먹통이 발생하지 않는다.
>
>- 요청을 블로킹 방식으로 처리하는 상황에서
>    - 요청 처리에 일정 시간이 필요하고, 스레드가 충분하지 못하다면, 일부 요청은 큐에서 쌓여 대기하다가 시차를 두고 처리된다.
>    - 큐에 쌓여 대기하는 요청을 줄이려면 스레드를 늘리는 수 밖에 없다.

결국 제한된 자원 상황에서 많은 요청을 처리하려면 스레드를 늘리는 수 밖에 없지만, 자체 스택을 갖고 있는 스레드를 많이 사용하면 메모리 한계에 부딪힐 수 있고, 많은 스레드는 많은 Context Switching 을 유발해서 성능 저하의 원인이 되기도 한다.

그렇다면 스레드를 무작정 늘리기보다는 스레드 사용 효율을 높이는 방법을 찾아봐야 할 것 같다.

3편에서는 스레드 효율을 높이는 방법으로 NIO에 대해 알아본다.
