# Back to the Essence - Java Servers - 2편

1편에서 블로킹 방식의 싱글 스레드 소켓 서버를 만들어봤고 다음의 문제가 있음을 발견했다.

>- 블로킹 방식의 싱글 스레드 소켓 서버는 시간 끄는 이상한 클라이언트가 하나만 들어와도 서버가 먹통이 되고, 다른 클라이언트까지 먹통될 수 있다.

이제 시간 끄는 이상한 클라이언트가 들어오더라도 서버나 다른 클라이언트가 먹통이 되지 않도록 개선해야 한다.

가장 간단한 방법은 클라이언트의 요청마다 별개의 스레드에서 처리하게 하는 것이다. 가장 널리 사용하는 방식이며 서블릿도 이 방식에 기반을 두고 있고, 서블릿에 기반을 둔 Spring MVC도 이 방식이다.

# Classic IO - Multi Thread ServerSocket

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
//                    Utils.sleep(50L);
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

1편에서 문제를 유발했던 시나리오 그대로 수행해보자.

1. EchoSocketServerMultiThread 실행
1. EchoSocketClient 실행, EchoSocketClient는 연결 요청 후 5초 후에 메시지를 서버에 전송
1. 5초 이내에 다른 터미널에서 `echo -n '아무거나' | nc localhost 7777` 실행해서 메아리가 터미널에 바로 찍히면 문제 해결

실제로는 실습 편의를 위해 EchoSocketClient가 5초가 아니라 1분 후에 보내도록 설정했다. 결과는 다음 움짤과 같이 여러 터미널 창에서 동시에 `echo -n '아무거나' | nc localhost 7777`를 실행해도 모두 거의 동시에 메아리가 출력됐다. 움짤 파일 크기를 작게하기 위해 4개의 터미널창만 캡쳐했지만 실제로는 12개의 창에서 동시에 요청을 날렸다.

![Imgur](https://i.imgur.com/HHYMsq0.gif)

로그는 다음과 같다.

```
scratchpad-server git:main 🍺🦑🍺🍕🍺 ❯ tail -f temp.log                                                                                               ✹
[SERVER -            main] 2020-11-02T01:36:54.427095 - ===============================
[SERVER -            main] 2020-11-02T01:36:54.446306 - Multi Thread Socket Echo Server 시작
[SERVER -            main] 2020-11-02T01:36:54.446730 - ---------------------------
[SERVER -            main] 2020-11-02T01:36:54.447015 - Echo Server 대기 중

[SERVER -            main] 2020-11-02T01:37:00.181527 - ---------------------------
[SERVER - pool-1-thread-1] 2020-11-02T01:37:00.181621 - Client 접속!!!   <== 1분 후 메시지 보내는 EchoSocketClient 요청 accept, 스레드: pool-1-thread-1
[SERVER -            main] 2020-11-02T01:37:00.181954 - Echo Server 대기 중
[SERVER - pool-1-thread-1] 2020-11-02T01:37:00.181981 - Echo 시작
[CLIENT -            main] 2020-11-02T01:37:00.197423 - Client 시작

[SERVER -            main] 2020-11-02T01:37:08.412302 - ---------------------------
[SERVER - pool-1-thread-2] 2020-11-02T01:37:08.412373 - Client 접속!!!   <== 여기서부터는 터미널 nc 클라이언트 요청 accept
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
[CLIENT -            main] 2020-11-02T01:38:00.227402 - 메시지 전송 시작   <== EchoSocketClient 가 1분 후 메시지 전송
[CLIENT -            main] 2020-11-02T01:38:00.238871 - 메시지 print 완료
[CLIENT -            main] 2020-11-02T01:38:00.242828 - 메시지 flush 완료
[CLIENT -            main] 2020-11-02T01:38:00.243060 - 서버 Echo 대기...
[CLIENT -            main] 2020-11-02T01:38:00.261370 - 서버 Echo 도착
[SERVER - pool-1-thread-1] 2020-11-02T01:38:00.261379 - Echo 완료  <== 첫 요청을 받았던 스레드 pool-1-thread-1가 계속 블록돼 있다가 이제서야 Echo 처리
[CLIENT -            main] 2020-11-02T01:38:00.263653 - 서버 Echo msg: Server Echo - 안녕, echo server
```
