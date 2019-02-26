# Java NIO Channel - ServerSocketChannel

- `ServerSocketChannel.open()`을 통해 Blocking 모드인 ServerSocketChannel을 생성
  - `configureBlocking(false)`로 Non-Blocking 모드로 설정 가능
- `bind(SocketAddress)`로 소켓에 바인딩
- `accept()`로 클라이언트로부터의 연결 요청을 수신하고 클라이언트와 연결되는 Blocking 모드인 SocketChannel을 생성해서 반환
- 반환된 SocketChannel과 Buffer를 통해 메시지 송수신


## ServerSocketChannel.accept()

`ServerSocketChannel.accept()`는 결국 ServerSocketChannelImpl 에 선언되어 있는 native 메서드인 `accept0()`를 호출해서 그 결과에 따라 null 또는 새로운 SocketChannel 을 반환

### ServerSocketChannel이 Blocking 모드일 때

- `accept0()`는 클라이언트 쪽에서 `SocketChannel.connect(SocketAddress)`을 호출할 때까지 값을 반환하지 않고 Blocking    
- 클라이언트 쪽에서 `SocketChannel.connect(SocketAddress)`을 호출하여 서버-클라이언트 간 소켓 연결이 성립하고 소켓에 대한 FileDescriptor가 생성
- FileDescriptor가 생성되면 Blocking하고 있던 `accept0()`는 1을 반환
- `accept0()`가 1을 반환하면 `ServerSocketChannel.accept()`는 Blocking 모드인 SocketChannel을 새로 생성해서 반환

### ServerSocketChannel이 Non-Blocking 모드일 때

- `accept0()`는 `IOStatus.UNAVAILABLE`(값은 -2)를 바로 반환하며, 결과적으로 `ServerSocketChannel.accept()`는 null을 바로 반환
    - 따라서 Non-Blocking 모드에서는 무한 루프로 `ServerSocketChannel.accept()`를 계속 호출해야 함
- 클라이언트 쪽에서 `SocketChannel.open()`을 호출하여 서버-클라이언트 간 소켓 연결이 성립하고 소켓에 대한 FileDescriptor가 생성
- FileDescriptor가 생성되면 무한 루프에 의해 새로 호출된 `accept0()`가 1을 반환
- `accept0()`가 1을 반환하면 `ServerSocketChannel.accept()`는 Blocking 모드인 SocketChannel을 새로 생성해서 반환


## SocketChannel.read(ByteBuffer)

`SocketChannel.read()`는 결국 SocketDispatcher 에 선언되어 있는 native 메서드인 `read0()`를 호출해서 그 결과에 따라 null 또는 실제 읽어들인 메시지를 Buffer에 저장

### SocketChannel이 Blocking 모드일 때

- `read0()`는 클라이언트 쪽에서 `SocketChannel.write(ByteBuffer)`을 호출할 때까지 값을 반환하지 않고 Blocking    
- 클라이언트 쪽에서 `SocketChannel.write(ByteBuffer)`을 호출하여 실제 데이터를 전송하면
- `accept0()`는 1을 반환
- `accept0()`가 1을 반환하면 `ServerSocketChannel.accept()`는 Blocking 모드인 SocketChannel을 새로 생성해서 반환

### ServerSocketChannel이 Non-Blocking 모드일 때

- `accept0()`는 `IOStatus.UNAVAILABLE`(값은 -2)를 바로 반환하며, 결과적으로 `ServerSocketChannel.accept()`는 null을 바로 반환
    - 따라서 Non-Blocking 모드에서는 무한 루프로 `ServerSocketChannel.accept()`를 계속 호출해야 함
- 클라이언트 쪽에서 `SocketChannel.open()`을 호출하여 서버-클라이언트 간 소켓 연결이 성립하고 소켓에 대한 FileDescriptor가 생성
- FileDescriptor가 생성되면 무한 루프에 의해 새로 호출된 `accept0()`가 1을 반환
- `accept0()`가 1을 반환하면 `ServerSocketChannel.accept()`는 Blocking 모드인 SocketChannel을 새로 생성해서 반환



