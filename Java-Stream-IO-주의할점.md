# Java Stream I/O 주의할 점

자바에는 입출력 스트림 구현체가 굉장히 다양하다. 이 중에서 주의할 점을 짚어본다.

# DataInputStream, DataOutputStream

자바의 primitive 데이터 타입을 지원하는 이 두 클래스를 사용하면 자바 서버, 클라이언트 끼리는 통신이 잘 되지만, CLI 클라이언트 처럼 자바 외의 클라이언트와는 통신이 잘 되지 않으며, EOFException 이 발생할 수 있다.

따라서 **DataInputStream, DataOutputStream 두 클래스는 자바 서버/클라이언트끼리 통신할 때만 사용해야 한다.**

# BufferedReader

스트림 데이터를 버퍼를 통해 읽어오는 데 널리 사용되는 방식이다.

**`BufferedReader.readLine()` 으로 데이터를 읽을 때 newline 을 기준으로 데이터를 읽으므로, 데이터를 전송할 때 반드시 newline 을 넣어줘야 한다.**

그렇지 않으면 데이터 읽기가 완료되지 않아서, blocking 방식으로 작성된 서버의 경우 연결된 클라이언트에서 접속을 끊거나 newline 을 추가로 보내지 않으면 계속 blocking 된 상태로 남아서 다른 클라이언트의 요청을 받지 못할 수도 있다.

따라서 **클라이언트에서는 `BufferedWriter.write(data + System.lineSeparator())` 로 보내거나 아니면 `PrintWriter.println(data)` 로 보내야 한다.**
