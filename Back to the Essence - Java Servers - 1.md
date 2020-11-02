title: Back to the Essence - Java Servers - (1)
date: 2020-11-02 00:34:50
categories:
  - Network
tags:
  - Java
  - I/O
  - Java IO
  - Echo Server
  - ServerSocket
  - Socket
  - Blocking
  - netcat
  - accept
thumbnailImage: https://i.imgur.com/uUynZ6S.png
coverImage: cover-single-thread-server.png
---

# Back to the Essence - Java Servers - 1편

서버 프로그래밍을 한다고는 하지만, 지난 수년 간 굴러도 스프링 위에서만 구르다보니 스프링 없이는, 아니 이제는 스프링만으로도 뭘 못할 것 같고 스프링 부트 없이는 간단한 메아리(Echo) 서버조차 못 만드는 ~~경지~~지경에 이르렀다. 이 아니 부끄러운가..

그래서 Java가 제공해주는 classic IO, NIO, NIO2로 간단한 Echo Server를 만들어보면서 기본기를 좀 다져보려 한다.  
만드는 데서 그치지 않고 그동안 간접 경험으로만 알아왔던 NIO, NIO2 의 장단점을 부하테스트를 통해 확인해보고자 한다.  
나름 원대한 계획이지만 목표한 걸 모두 얻을 수 있을지는 미지수다. 그냥 달려보자.

# Client

서버를 호출할 클라이언트는 크게 3가지다.

- Java Socket Client
- nc(netcat)
- JMeter Client

이 중에서 코딩이 필요한 건 Java Socket Client 뿐이고 코드는 다음과 같다. 이해를 위해 로깅을 많이 넣었는데, 로깅 빼면 설명할 것도 없다.  
참고로 로깅을 콘솔이 아닌 temp.log 파일에 찍는다. 이유는 서버와 클라이언트의 로그를 한 군데 모아서 보는 게 이해하는 데 도움이 되기 때문이다.

```java
package io.homo_efficio.server.socket;

import io.homo_efficio.server.common.Constants;
import io.homo_efficio.server.common.Utils;

import java.io.*;
import java.net.Socket;

/**
 * @author homo.efficio@gmail.com
 * created on 2020-10-10
 */
public class EchoSocketClient {

    public static void main(String[] args) throws IOException {
        String message = "안녕, echo server";

        try (Socket clientSocket = new Socket(Constants.SERVER_HOST_NAME, Constants.SERVER_PORT);
             FileOutputStream fos = Utils.getCommonFileOutputStream();
             PrintWriter out = new PrintWriter(clientSocket.getOutputStream());
             BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()))
        ) {
            Utils.clientTimeStamp("Client 시작", fos);
            // Utils.sleep(5000L);  // 서버 blocking 확인 시 사용
            Utils.clientTimeStamp("메시지 전송 시작", fos);
            out.println(message);
            Utils.clientTimeStamp("메시지 print 완료", fos);
            out.flush();
            Utils.clientTimeStamp("메시지 flush 완료", fos);
            Utils.clientTimeStamp("서버 Echo 대기...", fos);
            // in.readLine() 은 읽을 데이터가 들어올 때까지 blocking 이므로 while (true) 불필요
            String messageFromServer = in.readLine();
            Utils.clientTimeStamp("서버 Echo 도착", fos);
            Utils.clientTimeStamp("서버 Echo msg: " + messageFromServer, fos);
        }
    }
}

```


# Classic IO - Single Thread ServerSocket

이제 서버를 만들어 보자. 1번 타자는 Classic IO(또는 BIO(Blocking IO))로 만든 울트라 심플 싱글 스레드 소켓 서버다.

```java
package io.homo_efficio.server.socket;

import io.homo_efficio.server.common.Constants;
import io.homo_efficio.server.common.EchoProcessor;
import io.homo_efficio.server.common.Utils;

import java.io.FileOutputStream;
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * @author homo.efficio@gmail.com
 * created on 2020-10-10
 */
public class EchoSocketServerSingleThread {

    public static void main(String[] args) throws IOException {
        EchoSocketServerSingleThread echoSocketServerSingleThread = new EchoSocketServerSingleThread();
        echoSocketServerSingleThread.start();
    }

    public void start() throws IOException {
        try (ServerSocket serverSocket = new ServerSocket(Constants.SERVER_PORT);
             FileOutputStream fos = Utils.getCommonFileOutputStream()
        ) {
            Utils.serverTimeStamp("===============================", fos);
            Utils.serverTimeStamp("Echo Server 시작", fos);

            while (true) {
                Utils.serverTimeStamp("---------------------------", fos);
                Utils.serverTimeStamp("Single Thread Socket Echo Server 대기 중", fos);
                // accept() 는 연결 요청이 올 때까지 return 하지 않고 blocking
                Socket acceptedSocket = serverSocket.accept();

                // 연결 요청이 오면 accept() 가 반환하고 요청 처리 로직 수행
                Utils.serverTimeStamp("Client 접속!!!", fos);
//            Utils.sleep(50L);
                Utils.serverTimeStamp("Echo 시작", fos);
                EchoProcessor.echo(acceptedSocket);
                Utils.serverTimeStamp("Echo 완료", fos);
            }
        }
    }
}

```

`ServerSocket`으로 서버 소켓을 생성하고, `accept()`로 클라이언트의 연결을 기다리고, 연결이 오면 클라이언트에게 메시지를 메아리로 되돌려 준다.

메아리를 담당하는 EchoProcessor는 다음과 같다.

```java
public abstract class EchoProcessor {

    private static final FileOutputStream fos = Utils.getCommonFileOutputStream();

    public static void echo(Socket socket) throws IOException {
        try (BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
             PrintWriter out = new PrintWriter(socket.getOutputStream())
        ) {
            String clientMessage = in.readLine();  // in에 읽을 게 들어올 때까지 blocking
            String serverMessage = "Server Echo - " + clientMessage + System.lineSeparator();
            out.println(serverMessage);
            out.flush();
        }
    }
}
```

`Socket`을 인자로 받고, 소켓에서 Reader, Writer를 뽑아내서, Reader에서 메아리를 읽고 'Server Echo -'라는 문자열을 앞에 붙여서 Writer로 회신한다.

여기서 주의할 점이 있다. **서버가 보내는 메시지에 비어 있는 행이 포함돼야 클라이언트가 `readLine()`으로 읽을 때 행을 구별해서 문제 없이 읽고 출력할 수 있다.** 비어 있는 행이 없으면 클라이언트의 `readLine()`이 계속 비어 있는 행을 기다리면서 서버와의 연결을 점유하게 되고, 싱글 스레드인 서버는 먹통 상태가 된다.

# 실습

1. EchoSocketServerSingleThread 를 실행하고, EchoSocketClient 를 실행하면 temp.log 파일에 다음과 같이 로그가 찍한다.

    ```
    [SERVER -            main] 2020-11-01T23:49:25.119684 - ===============================
    [SERVER -            main] 2020-11-01T23:49:25.133603 - Echo Server 시작
    [SERVER -            main] 2020-11-01T23:49:25.133994 - ---------------------------
    [SERVER -            main] 2020-11-01T23:49:25.134174 - Single Thread Socket Echo Server 대기 중
    [SERVER -            main] 2020-11-01T23:49:26.976560 - Client 접속!!!
    [SERVER -            main] 2020-11-01T23:49:26.976861 - Echo 시작
    [CLIENT -            main] 2020-11-01T23:49:26.992329 - Client 시작
    [CLIENT -            main] 2020-11-01T23:49:27.006950 - 메시지 전송 시작
    [CLIENT -            main] 2020-11-01T23:49:27.007250 - 메시지 print 완료
    [CLIENT -            main] 2020-11-01T23:49:27.008839 - 메시지 flush 완료
    [CLIENT -            main] 2020-11-01T23:49:27.009160 - 서버 Echo 대기...
    [CLIENT -            main] 2020-11-01T23:49:27.020318 - 서버 Echo 도착
    [SERVER -            main] 2020-11-01T23:49:27.021049 - Echo 완료
    [SERVER -            main] 2020-11-01T23:49:27.021302 - ---------------------------
    [SERVER -            main] 2020-11-01T23:49:27.021471 - Single Thread Socket Echo Server 대기 중
    [CLIENT -            main] 2020-11-01T23:49:27.021674 - 서버 Echo msg: Server Echo - 안녕, echo server
    ```
2. 서버는 여전히 대기 중이므로 다른 터미널에서 `echo -n '아무거나' | nc localhost 7777`을 입력하면 다음과 같이 Echo 메시지가 바로 출력되어 나온다.

    ```
    다른 터미널창
    🍺🦑🍺🍕🍺 ❯ echo -n '아무거나' | nc localhost 7777                                             
    Server Echo - 아무거나
    ```

    ```
    ... 윗 부분 생략 ...
    [SERVER -            main] 2020-11-01T23:49:27.021302 - ---------------------------
    [SERVER -            main] 2020-11-01T23:49:27.021471 - Single Thread Socket Echo Server 대기 중


    [SERVER -            main] 2020-11-02T00:03:47.863975 - Client 접속!!!
    [SERVER -            main] 2020-11-02T00:03:47.874942 - Echo 시작
    [SERVER -            main] 2020-11-02T00:03:47.878276 - Echo 완료
    [SERVER -            main] 2020-11-02T00:03:47.878572 - ---------------------------
    [SERVER -            main] 2020-11-02T00:03:47.878849 - Single Thread Socket Echo Server 대기 중
    ```

3. EchoSocketClient 에서 `// Utils.sleep(5000L);  // 서버 blocking 확인 시 사용`라고 돼 있던 부분의 주석을 해제하고 실행해서 클라이언트가 서버와 연결된 후 5초 후에 서버에 메시지를 전송하도록 하고, 5초 안에 다른 터미널에서 `echo -n '아무거나' | nc localhost 7777`을 입력한다.  

    - 그러면 메아리가 터미널에 금방 출력되지 않고 5초 후에 출력된다.
    - 이유는 앞서 말한 것처럼 EchoSocketClient가 5초 후에 메시지를 보내는 동안, EchoProcessor의 `in.readLine()`이 블로킹 상태로 대기하는데, 서버의 스레드도 1개 뿐이라 다른 요청을 `accept()` 할 수 없기 때문이다.
    - 그래서 터미널 클라이언트도 5초간 블로킹 상태로 대기하게 된다.
    - 결국 **이상한 클라이언트가 하나 끼면 서버도 먹통되고 다른 클라이언트까지 먹통이 전파될 수 있다.**


# 정리

>- 블로킹 방식의 싱글 스레드 소켓 서버는 시간 끄는 이상한 클라이언트가 하나만 들어와도 서버가 먹통이 되고, 다른 클라이언트까지 먹통될 수 있다.

이 문제는 어떻게 해결할까? 2편에서 알아보자.
