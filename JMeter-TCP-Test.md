# JMeter TCP Test

## JMeter 설치 및 실행

- 설치: https://jmeter.apache.org/download_jmeter.cgi 에서 파일 받아 압축 해제
- 실행: bin/jmeter 실행

## TCP 테스트 

- 아래 그림 좌측 같이 Thread Group, View Results Tree, TCP Sampler 를 추가
- TCP Sampler 에서 아래 그림과 같이 설정
- Text to send 내용에 공백 라인 반드시 추가

  ![Imgur](https://i.imgur.com/nkCdaT1.png)

- 테스트 결과

  ![Imgur](https://i.imgur.com/xrFtipU.png)

  ![Imgur](https://i.imgur.com/OKsNQTw.png)

- Text to send 내용에 공백 라인 추가 하지 않으면 서버로 전송을 하지 않으며,
  - Response Timeout 설정을 안 해두면 계속 먹통,
  - Response Timeout 설정을 해두면 서버로부터 응답은 받지 못해 bytes read 가 0인 채로 Response Timeout 되고,
  - Timeout 되면서 버퍼에 있던 데이터가 비정상적으로 서버에 전송되고 클라이언트에 echo 하지만, 클라이언트는 이미 Timeout 된 상태

    ```
    2020-10-31 11:59:27,443 INFO o.a.j.e.StandardJMeterEngine: Running the test!
    2020-10-31 11:59:27,444 INFO o.a.j.s.SampleEvent: List of sample_variables: []
    2020-10-31 11:59:27,444 INFO o.a.j.g.u.JMeterMenuBar: setRunning(true, *local*)
    2020-10-31 11:59:27,480 INFO o.a.j.e.StandardJMeterEngine: Starting ThreadGroup: 1 : Thread Group
    2020-10-31 11:59:27,480 INFO o.a.j.e.StandardJMeterEngine: Starting 1 threads for group Thread Group.
    2020-10-31 11:59:27,480 INFO o.a.j.e.StandardJMeterEngine: Thread will continue on error
    2020-10-31 11:59:27,480 INFO o.a.j.t.ThreadGroup: Starting thread group... number=1 threads=1 ramp-up=1 delayedStart=false
    2020-10-31 11:59:27,481 INFO o.a.j.t.ThreadGroup: Started thread group number 1
    2020-10-31 11:59:27,481 INFO o.a.j.e.StandardJMeterEngine: All thread groups have been started
    2020-10-31 11:59:27,481 INFO o.a.j.t.JMeterThread: Thread started: Thread Group 1-1
    2020-10-31 11:59:27,481 INFO o.a.j.p.t.s.TCPClientImpl: Using platform default charset:UTF-8
    2020-10-31 11:59:28,486 INFO o.a.j.t.JMeterThread: Thread is done: Thread Group 1-1
    2020-10-31 11:59:28,486 INFO o.a.j.t.JMeterThread: Thread finished: Thread Group 1-1
    2020-10-31 11:59:28,487 INFO o.a.j.e.StandardJMeterEngine: Notifying test listeners of end of test
    2020-10-31 11:59:28,487 INFO o.a.j.g.u.JMeterMenuBar: setRunning(false, *local*)
    2020-10-31 12:12:53,414 INFO o.a.j.e.StandardJMeterEngine: Running the test!
    2020-10-31 12:12:53,419 INFO o.a.j.s.SampleEvent: List of sample_variables: []
    2020-10-31 12:12:53,420 INFO o.a.j.g.u.JMeterMenuBar: setRunning(true, *local*)
    2020-10-31 12:12:53,507 INFO o.a.j.e.StandardJMeterEngine: Starting ThreadGroup: 1 : Thread Group
    2020-10-31 12:12:53,507 INFO o.a.j.e.StandardJMeterEngine: Starting 1 threads for group Thread Group.
    2020-10-31 12:12:53,507 INFO o.a.j.e.StandardJMeterEngine: Thread will continue on error
    2020-10-31 12:12:53,507 INFO o.a.j.t.ThreadGroup: Starting thread group... number=1 threads=1 ramp-up=1 delayedStart=false
    2020-10-31 12:12:53,507 INFO o.a.j.t.ThreadGroup: Started thread group number 1
    2020-10-31 12:12:53,507 INFO o.a.j.e.StandardJMeterEngine: All thread groups have been started
    2020-10-31 12:12:53,508 INFO o.a.j.t.JMeterThread: Thread started: Thread Group 1-1
    2020-10-31 12:12:53,508 INFO o.a.j.p.t.s.TCPClientImpl: Using platform default charset:UTF-8
    2020-10-31 12:12:56,511 ERROR o.a.j.p.t.s.TCPSampler: 
    org.apache.jmeter.protocol.tcp.sampler.ReadException: Error reading from server, bytes read: 0
      at org.apache.jmeter.protocol.tcp.sampler.TCPClientImpl.read(TCPClientImpl.java:122) ~[ApacheJMeter_tcp.jar:5.3]
      at org.apache.jmeter.protocol.tcp.sampler.TCPSampler.sample(TCPSampler.java:398) [ApacheJMeter_tcp.jar:5.3]
      at org.apache.jmeter.threads.JMeterThread.doSampling(JMeterThread.java:630) [ApacheJMeter_core.jar:5.3]
      at org.apache.jmeter.threads.JMeterThread.executeSamplePackage(JMeterThread.java:558) [ApacheJMeter_core.jar:5.3]
      at org.apache.jmeter.threads.JMeterThread.processSampler(JMeterThread.java:489) [ApacheJMeter_core.jar:5.3]
      at org.apache.jmeter.threads.JMeterThread.run(JMeterThread.java:256) [ApacheJMeter_core.jar:5.3]
      at java.lang.Thread.run(Thread.java:832) [?:?]
    Caused by: java.net.SocketTimeoutException: Read timed out
      at sun.nio.ch.NioSocketImpl.timedRead(NioSocketImpl.java:283) ~[?:?]
      at sun.nio.ch.NioSocketImpl.implRead(NioSocketImpl.java:309) ~[?:?]
      at sun.nio.ch.NioSocketImpl.read(NioSocketImpl.java:350) ~[?:?]
      at sun.nio.ch.NioSocketImpl$1.read(NioSocketImpl.java:803) ~[?:?]
      at java.net.Socket$SocketInputStream.read(Socket.java:982) ~[?:?]
      at java.io.InputStream.read(InputStream.java:218) ~[?:?]
      at org.apache.jmeter.protocol.tcp.sampler.TCPClientImpl.read(TCPClientImpl.java:105) ~[ApacheJMeter_tcp.jar:5.3]
      ... 6 more
    2020-10-31 12:12:56,512 INFO o.a.j.t.JMeterThread: Thread is done: Thread Group 1-1
    2020-10-31 12:12:56,512 INFO o.a.j.t.JMeterThread: Thread finished: Thread Group 1-1
    2020-10-31 12:12:56,512 INFO o.a.j.e.StandardJMeterEngine: Notifying test listeners of end of test
    2020-10-31 12:12:56,513 INFO o.a.j.g.u.JMeterMenuBar: setRunning(false, *local*)

    ```

- TCP Sampler 관련 자세한 내용은 https://jmeter.apache.org/usermanual/component_reference.html#TCP_Sampler 참고


