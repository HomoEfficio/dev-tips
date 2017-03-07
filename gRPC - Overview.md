# gRPC

- gRPC가 뭐냐?
    - **g**RPC **R**emote **P**rocedure **C**all
        - GNU is Not Unix 아재
    - RPC가 뭐냐? 를 알아보기 전에.. gRPC가 누구냐?
        - Stubby의 오픈소스 버전
            - Stubby는 뭐냐?
                - 구글은 1주일에 20억개의 컨테이너를 런칭
                - 컨테이너들끼리의 어떻게 통신하나?
                    - borg: 클러스터 관리 담당 -> Kubernetes
                    - stubby: 통신 담당 -> gRPC(1.0.0 in 2016.08)
        - RPC의 정의
        - 입관되신 분들: RMI, CORBA, SOAP
        - 왜 입관? REST(또는 RESTful API) + JSON

- RESTful API가 있는데 뭐하러 gRPC?
    - gRPC 특징
        - Binary
        - HTTP/2
        - Polyglot
    - gRPC 비교 우위 over RESTful API

- 어떻게 쓰는거냐?
    - Remote XX call
        - 호출 대상의 식별
        - 메시지를 전송하는 방법
        - 데이터의 종류
    - 예제
        - raw gRPC
            - unary - unary
            - unary - streaming
            - streaming - unary
            - streaming - streaming
        - 스프링 부트 rest api vs 스프링 부트 grpc
            - springboot grpc
                - https://github.com/LogNet/grpc-spring-boot-starter
                - https://www.versioneye.com/java/org.lognet:grpc-spring-boot-starter/0.0.2

- 실무 적용 고려 사항
    - Versioning updating client
    - Service discovery
    - authentication
    - authorization
    - exception handling
    - failure control

- Microservice와의 관계
    - good fit for server-to-server call
- Reactive Programming과의 관계
    - onComplete, onError 등 reactive에서 따옴
    - 


----

- Protocol buffer
    - Adv over XML
        - are simpler
        - are 3 to 10 times smaller
        - are 20 to 100 times faster
        - are less ambiguous
        - generate data access classes that are easier to use programmatically
    - XML over protobuf
        - Good for markup based text doc modeling
        - human readable, editable
        - self-describing
            - 근데 XML도 dtd나 xsd가 있어야 명확하긴 하다.


- RESTful API가 있는데 뭐하러 gRPC?
    - 성능
        - http://bit.ly/etcd_grpc
        - marshalling/unmarshalling vs serialization/deserialization
            - https://en.wikipedia.org/wiki/Marshalling_(computer_science)
            - java에서는 xml <-> java obj는 marshal, json <-> java obj는 serialization이라는 표현 사용
    - Microservices와의 궁합
        - polyglot
            - 언어별 성능: https://performance-dot-grpc-testing.appspot.com/explore?dashboard=5712453606309888
        - server to server calls

- 어떻게 쓰는거냐?
    - 호출 대상의 식별
    - 메시지를 전송하는 방법
    - 데이터의 종류
    - 예제
    - 스프링 부트 rest api vs 스프링 부트 grpc
    - springboot grpc
        - https://github.com/LogNet/grpc-spring-boot-starter
        - https://www.versioneye.com/java/org.lognet:grpc-spring-boot-starter/0.0.2

- 실무 적용 고려 사항
    - Versioning updating client
    - Service discovery

- https://www.youtube.com/watch?v=-2sWDr3Z0Wo
- gRPC over HTTP/2 vs gRPC over WebSockets?
    - 35:00
    - 내용 잘 모르겠음
- Versioning updating client
    - 35:40
    - app에서는 어렵지만 서버 투 서버에서는 protobuf가 backward compatibility가 있으므로, 정의만 변하지 않는다면 구버전 client가 신버전 server를 호출해도 깨지지 않으므로 서비스를 내리고 업데이트 할 필요가 없다.
- load balancing
    - 38:47
    - 클라이언트가 hw loadbalancer에 호출하는 방식 가능
    - client side loadbalancing 지원?
- client side software loadbalancer 예제?
    - 40:36
    - Netflix에 있을 것이다.
- channel security support?
    - 42:07
    - SSL 지원한다. SSL은 양방향 보안도 지원한다. client나 server에 custom interceptor를 추가해서 ACL 적용도 가능하다.
- WebSocket와 HTTP/2의 성능 차이?
    - 43:30
    - websocket은 connection을 multiplex 할 수 없다. 따라서 websocket은 연결을 더 많이 열어야 할 필요가 있다.

- Unary RPC: the client sends a request to the server, the server sends a response
- Streaming RPC: the client and server may each send one or more messages


# https://www.youtube.com/watch?v=UOIJNygDNlE

- 구글은 1주일에 20억개의 컨테이너를 런칭
- 어거 어떻게 서로 살아있는지 확인하고 통신하나?
- borg: google's container orchestration scheduling service
    - 이게 Kubernetes로 오픈
- stubby: the way these containers talk to each other
    - 이게 gRPC로 오픈

- the biggest issue in changing a monolith into microservices lies in changing the communication pattern - Martin Folwer

- monolith에서는 단순히 function call이었는데, microservice에서는 네트워크 호출이된다.
- 앱이 확장되면 네트워크 호출도 지수적으로 증가하는 문제
- gRPC가 good api design을 도와줄 수 있다.
- 구글에서는 초당 10^10(100억)의 rpc 발생
- y grpc is good for building microservice
- grpc is niche
- protocol buffer
    - 걍 binary json이라고 생각해도..
    - IDL
    - req, res
    - binary
    - http2
        - multiplexing
            - single tcp connection
        - bidirectional streaming
        - flow control
            - backpressure와 비슷
- polyglot
- mobile first
    - less overhead
- kubernetes와 함께 zero downtime update
- github.com/thesandlord/samples/tree/master/grpc-kubernetes-microservices


# Protocol Buffer

- https://developers.google.com/protocol-buffers/

# REST API

- http://blog.remotty.com/blog/2014/01/28/lets-study-rest/#crud

# 예제

## Go

- https://medium.com/@shijuvar/building-high-performance-apis-in-go-using-grpc-and-protocol-buffers-2eda5b80771b#.84pojc9kt

# 기타

- [gRPC 1.0 - Google Cloud Platform blog](https://cloudplatform.googleblog.com/2016/08/gRPC-a-true-Internet-scale-RPC-framework-is-now-1-and-ready-for-production-deployments.html)
- [yCombinator - gRPC](https://news.ycombinator.com/item?id=12344995)
- [gopherAcademy etcd - gRPC vs JSON](https://blog.gopheracademy.com/advent-2015/etcd-distributed-key-value-store-with-grpc-http2/)
- [GRPC vs REST on Node.js](https://albertsalim.xyz/grpc-vs-rest-node-js/)
- [gRPC gateway](https://github.com/grpc-ecosystem/grpc-gateway/blob/master/README.md)
- [Whats Wrong With Corba](http://wiki.c2.com/?WhatsWrongWithCorba)
- [JSON-RPC example](https://dzone.com/articles/is-java-remote-procedure-call-dead-in-the-rest-age)
- [SOAP is dead](http://serviceorientedarchitect.com/soap-is-dead-lessons-learned-from-soa-fying-a-monolith/)
- [SOAP-WSDL 코드 예](http://stackoverflow.com/questions/14541066/difference-between-a-soap-message-and-a-wsdl)
- [Java RMI 가이드](https://www.slideshare.net/sunnykwak90/java-rmi-49404122)
- [gRPC 언어별 unary/streaming 성능 비교](https://performance-dot-grpc-testing.appspot.com/explore?dashboard=5712453606309888)
- http://blog.appkr.kr/work-n-play/how-to-use-apache-thrift-in-php-part-1/
- http://bcho.tistory.com/1011
- http://fbsight.com/t/rpc-grpc/49819/5
- http://www.grpc.io/faq/
- [HTTP/2](http://www.popit.kr/%EB%82%98%EB%A7%8C-%EB%AA%A8%EB%A5%B4%EA%B3%A0-%EC%9E%88%EB%8D%98-http2/)
