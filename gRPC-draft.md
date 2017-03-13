# gㅏ벼운 RPC, gRPC in SpringCamp 2017

## 기

- gRPC가 뭐냐?
    - **g**RPC **R**emote **P**rocedure **C**all
        - GNU: **G**nu is **N**ot **U**nix (아재)
- RPC가 뭐냐?
    - 위키피디아 정의(한글로 풀기)
    
        > In distributed computing a remote procedure call (RPC) is when a computer program causes a procedure (subroutine) to execute in another address space (commonly on another computer on a shared network), which is coded as if it were a normal (local) procedure call, without the programmer explicitly coding the details for the remote interaction. That is, the programmer writes essentially the same code whether the subroutine is local to the executing program, or remote.
    - RPC 절차도
        - http://lycog.com/distributed-systems/remote-procedure-call/
    - 입관되신 분들..
        - CORBA
            - www.ejbtutorial.com/programming/tutorial-for-corba-hello-world-using-java
        - SOAP
            - 이기종 플랫폼 간 호환성은 좋지만, XML이 무거워서..
    - 잊혀진..
        - RMI
            - CORBA에 비해 매우 간단하지만 Java 끼리만..
            - 하지만 티안나게 사용되는.. JMX, VisualVM
- 패왕 등장 HTTP + REST + JSON
- 그래서 RPC는 망?
    - 아니다. 구현체가 좋지 않았을 뿐..
    - 더군다나 클라우드 환경에서는 효율적인 RPC가 반드시 필요
- 그래서 구글에서 RPC를 다시 만들었다.
    - Stubby라는 RPC 구현체를 만들어서
        - 1주일에 20억개의 컨테이너가 런칭되는 구글 환경에서
        - 1초에 100억건의 RPC를 처리
- gRPC는 Stubby의 오픈 소스 후계자

## 승

- gRPC가 HRJ(HTTP REST JSON)보다 나은점
    - binary protocol(Protocol Buffer 3)
        - text보다 더 적은 데이터 공간으로 처리 가능 -> 네트워크, 메모리 효율성 좋음
        - text보다 marshal/unmarshal 부하가 적으므로 CPU 더 적게 사용 -> CPU 효율성 좋음 
    - HTTP/2
        - connection multiplexing
            - 여러 자원 요청을 하나의 커넥션으로 처리 가능 -> 네트워크 효율성 좋음
        - client/server streaming 가능
            - websocket과의 비교
    - 네트워크, 메모리, CPU 효율성 좋음
        - https://www.youtube.com/watch?v=BOW7jd136Ok
            - 처리량: 7:25
            - 처리량CPU: 7:35
        - 클라우드 환경에서의 서버 간 대규모 통신에 적합
        - 자원이 빈약한 모바일(또는 IoT까지?)에 적합
        - 바이트 수, 호출 수, CPU 수 등으로 과금되는 클라우드 환경에서 비용 절감에 도움

## 전

- 구현 예제    
    - 원격 프로시저 호출의 동작 방식
        - http://lycog.com/distributed-systems/remote-procedure-call/ 의 그림에 IDL, protoc 추가
    - 작성 순서
        - IDL 작성 및 code Gen
        - 서버 코드
        - 클라이언트 코드
    - 채팅? 대규모 데이터 전송으로 성능 비교? SpringBoot 연동?
        - SpringBoot 상에서 용량 크고 재미있는 다수의 gif 파일 전송 비교
            - 웃프다 gif
            - 김연아-세종대왕-민망 gif
            - 송중기 gif
            - 김희선-강호동-아는형님 gif

- 실무 적용 고려 사항(다는 아니고 몇 개만..)
    - Versioning updating client
    - Service discovery
    - authentication
    - authorization
    - exception handling
    - failure control

## 결

- gRPC 정리
- Microservice와의 관계
- Reactive Programming과의 관계
- 관련 자료 링크
