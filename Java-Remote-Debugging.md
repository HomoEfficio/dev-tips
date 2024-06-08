# Java Remote Debugging

Java 애플리케이션은 [Java Platform Debugger Architecture, JDPA](https://docs.oracle.com/en/java/javase/22/docs/specs/jpda/jpda.html)를 통해 디버깅 할 수 있다.

로컬 IDE에서 사용하는 디버거도 JDPA를 바탕으로 동작한다.

JDPA를 자세히 알고 싶으면 https://docs.oracle.com/en/java/javase/22/docs/specs/jpda/architecture.html 를 참고하고, 여기에서는 실무적으로 원격 디버깅을 사용하는 데 필요한 최소한의 정보만 정리한다.

## 디버깅 대상 애플리케이션 실행 설정 및 실행

java 명령 실행 시 다음 VM 옵션을 추가한다.

>-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8282

옵션의 대략 다음과 같고 자세한 내용은 https://docs.oracle.com/en/java/javase/22/docs/specs/jpda/conninv.html 를 참고한다.

- `transport=dt_socket`: dt는 debug transport 의 약자인 것 같고 소켓 방식으로 디버거와 디버깅 대상 VM이 통신한다는 것을 의미. socket 방식 말고 공유 메모리 방식도 있지만, 실무적으로 대부분 `dt_socket`로 지정한다.
- `server=y`: 디버거가 서버 모드(`address` 옵션에서 지정한 IP:PORT에서 listen)로 동작한다. `n`으로 지정하면 `address` 옵션에서 지정한 별도의 다른 디버거 애플리케이션에 클라이언트로서 접근한다. 실무적으로 대부분 디버깅 대상 애플리케이션이 실행될 때 `-agentlib`을 지정해서 디버거를 실행하므로 `y`로 지정한다.
- `suspend=n`: 디버거가 먼저 실행 완료될 때까지 디버깅 대상 애플리케이션의 실행을 보류한다. 디버깅 대상 애플리케이션의 로딩 과정을 디버깅해야 할 때만 `y`로 지정하고, 그 외 대부분의 상황에서는 `n`으로 지정한다.
- `address:*:8282`: `8282` 대신 원하는 포트번호를 지정하면 된다. `server=y`이면 `address`에서 지정한 IP:PORT에서 listen 하고, `server=n`이면 `address`에서 지정한 IP:PORT에 접속한다. 위에 `server=y`에서 설명한 것처럼 실무적으로는 대부분 `server=y`로 지정한다.
  - IP 번호를 `*`로 지정하면 보안 위험이 있으므로 실제 운영 서비스에서 원격 디버그를 하는 것은 권장하지 않는다. `allow`로 IP를 세분하게 지정할 수도 있으며 위 문서를 참고한다.

로컬에서는 굳이 Remote Debug 를 사용할 일이 없지만 필요하다면 IntelliJ 같은 IDE 에서 아래와 같이 VM options 를 추가하고 입력한다.

![Imgur](https://i.imgur.com/i08pYq8.png)

## 리모드 디버거 실행 설정 및 실행

리모트 디버거를 실행하기 전에 반드시 필요한 사전 조건은 원격에서 실행에 사용된 소스 코드와 로컬에 있는 소스 코드가 동일해야 한다는 것이다. 그래야 로컬 IDE에서 지정한 브레이크포인트 위치가 원격에서도 동일한 위치에 브레이크포인트로 지정된다. 소스가 다르면 원하지 않는 위치에서 브레이크포인트가 적용될 수 있다.

소스를 맞췄으면 다음과 같이 Run/Debug Configurations 에서 Remote JVM Debug 를 선택해서 설정 이름, 호스트, 포트를 지정한다. 여기서 지정한 포트는 위 디버깅 대상 애플리케이션 실행 시 지정한 포트와 동일해야 한다.

![Imgur](https://i.imgur.com/H8QEG4q.png)

로컬 IDE에서 리모드 디버거를 실행하면 다음과 같은 화면이 뜬다. 로컬에서 Run 모드로 실행한 애플리케이션에 접속하므로 localhost로 표시되었고, 포트는 8383으로 지정했다.

![Imgur](https://i.imgur.com/wSXpQrD.png)

리모트 디버그 실행이 성공했으면 로컬 IDE에서 디버깅 할 때와 마찬가지로 브레이크포인트를 지정하고,  
애플리케이션에 특정 행위를 가해서 지정한 지점이 애플리케이션에서 실행되도록 하면,
예를 들어 API 서버인 경우 브레이크포인트가 지정된 API를 호출하면 API 서버의 실행이 멈추고,
로컬 IDE에서도 브레이크포인트 위치에서 실행이 멈춘 것이 표시되며 로컬 디버거에서 하던 Step Into 등을 똑같이 원격에서 실행되는 애플리케이션 대상으로 수행할 수 있다.

브레이크포인트에서 멈추면 API 서버의 실행이 멈추므로 API 서버는 다른 요청도 처리하지 못하게 되므로 앞서 말한 것처럼 **실제 운영 서버를 대상으로는 리모트 디버그를 절대 사용해서는 안 된다.**





