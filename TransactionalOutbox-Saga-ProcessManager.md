# Transactional Outbox Pattern, Saga Pattern, Process Manager Pattern

[도메인 주도 설계 첫걸음](https://wikibook.co.kr/lddd/)(원서:
[Learning Domain Driven Design](https://www.amazon.com/Learning-Domain-Driven-Design-Aligning-Architecture/dp/1098100131/))의 '9장 커뮤니케이션 패턴 > 애그리것(책에는 애그리게이트라고 나와있으나 잘못된 발음) 연동'에 '아웃박스, 사가, 프로세스 관리자'가 한 데 묶여서 나온다.

셋의 목적이나 용도가 보는 관점에 따라 비슷하기도 다르기도 한데 한 곳에 묶어 놓으니 살짝 혼동이 있는 것 같아 맞는지 틀리는지 모르지만 나름 구분하고 비교해서 정리해본다.


## Transactional Outbox Pattern

[마이크로서비스 아키텍처 사이트](https://microservices.io/patterns/data/transactional-outbox.html)에 잘 정리 돼 있지만 영어.. [강남언니 블로그 자료](https://blog.gangnamunni.com/post/transactional-outbox/)에 우리말로 친절하게 잘 정리돼 있다.

Transactional Outbox Pattern(타이핑 하기 귀찮으니 앞으로 TOP라 한다)은 한 마디로 정리하면 **메시지 발행의 일관성을 보장**하는 데 사용하는 패턴이다.

마이크로서비스 아키텍처 사이트에 있는 아래 그림을 보면 Message Broker에서 끝나고 있다.

![](https://microservices.io/i/patterns/data/ReliablePublication.png)

강남언니 블로그에도 잘 나와 있는 것처럼 TOP는 다음과 같은 문제를 해결해준다.

- 발행돼야 할 메시지가 발행되지 않는 문제
- 발행되지 않아야 할 메시지가 발행되는 문제
- 메시지 순서가 꼬이는 문제

마이크로서비스 아키텍처 사이트 그림과 마찬가지로 아래 그림과 같이 Message Broker에서 끝나고 있다.

![](https://static.blog.gangnamunni.com/files/da/da3db100-873d-44a2-81bd-ba57e60d2a5d.jpeg)

정리하면 다음과 같다.

>결국 TOP는
>- 일관성 문제를 해결해주는 데
>- 그 범위가 '메시지 브로커에게 메시지를 발행하는 작업까지'다.

강남언니 블로그의 마지막에 Idempotent Handling을 다루면서 아래 그림이 나오는데, 

![](https://static.blog.gangnamunni.com/files/93/93f6ad37-0767-4275-9524-df49596207fc.jpeg)

멱등성 처리는 블로그글을 참고하고, 이 그림을 다른 관점에서 이용해보려 한다.

만약에 Search Message Handling Server가 Product Registered 메시지를 가져와서 Product 데이터를 Search DB에 저장하려는데, 비즈니스 로직에 맞지 않아서든 예외 때문이든 Search DB에 저장이 되지 않고 처리가 종료됐다고 생각해보자.

이렇게 되면 Product가 MarketDB에는 저장이 됐는데 SearchDB에는 저장되지 않아 해당 Product가 검색이 되지 않는 상황이 발생한다.
즉, Product Registered라는 메시지는 TOP에 의해 일관성 있게 메시지 브로커에 전송이 됐지만, MarketDB와 SearchDB 사이의 일관성은 깨져버렸다.

이처럼 Bounded Context 사이에 일관성이 깨지는 문제는 TOP만으로 해결할 수 없다. 이 문제를 해결하는 데 사용하는 것이 바로 사가(Saga) 패턴이다.


## Saga Pattern

사가 패턴은 한 마디로 정리하면 **여러 Bounded Context 사이의 일관성을 보장**하는 데 사용하는 패턴이다.

각각 다른 Bounded Context 사이에서 롤백을 할 수 없는데 일관성을 어떻게 보장할 수 있을까? 롤백 대신에 보상 트랜잭션(compensation transaction)을 활용해서 일관성을 보장한다.

예를 들어 BoundedContextA -> BoundedContextB -> BoundedContextC -> BoundedContextD 에 걸쳐 처리되는 작업이 있을 때 D까지 온 상태라면 A, B, C는 이미 커밋이 된 상태이다. 그런데 D의 비즈니스 로직에 맞지 않거나 또는 D에서 오류가 발생해서 D의 커밋이 완료되지 않는다면 이미 커밋된 A, B, C의 상태를 되돌려야 한다. 이미 커밋됐으므로 롤백을 할 수는 없고 A, B, C 각자의 상태값을 커밋 전의 상태값과 동일하게 변경하는 별도의 트랜잭션이 실행돼야 하는데 이를 보상 트랜잭션이라고 부른다.

다시 정리하면 사가 패턴은 **보상 트랜잭션을 사용해서 여러 Bounded Context 사이의 일관성을 보장**하는 패턴이다.

개인적으로는 그냥 Compensation Transaction Pattern이라고 이름지었다면 훨씬 금방 이해할 수 있었을 것 같다. 암튼 사가 패턴의 핵심은 보상 트랜잭션이고 이 보상 트랜잭션이 실행되는 방식에 따라 Choreography 방식과 Orchestration 방식으로 구분할 수 있다.

### Choreography

코레오그래피 방식은 Message Broker를 중심으로 자율적으로 동작한다.

![](https://learn.microsoft.com/ko-kr/azure/architecture/reference-architectures/saga/images/choreography-pattern.png)

A, B, C, D에서 보상 트랜잭션 실행이 필요한 상황 발생 시 이벤트를 발행하고, 이벤트를 구독하고 있는 쪽에서 보상 트랜잭션을 실행한다.

따라서 A, B, C, D가 서로에 대해 알 필요가 없고 이벤트를 발행하고 구독하기만 하면 되는 느슨한 구조를 가질 수 있다. 이렇게 말하면 좋기만 한 것 같은데 그럴리가.. 느슨하다보니 A -> B -> C -> D로 진행된다는 사실마저 파악하기 쉽지 않다. 따라서 A -> B -> C -> D로 진행된다는 사실을 굳이 알 필요 없거나, 쉽게 파악할 수 있을 정도로 간단한 상황에 적합하다.

https://learn.microsoft.com/ko-kr/azure/architecture/reference-architectures/saga/saga#choreography 에 장단점이 잘 정리돼 있다.

Message Broker를 중심으로 동작하는 코레오그래피 방식에서는 메시지 발행 일관성이 유지돼야 한다. 메시지 발행 일관성? 어디서 들어본 말 같은데.. TOP아냐? 그렇다. **코레오그래피에서는 사실상 TOP도 함께 사용돼야 한다.**

### Orchestration

오케스트레이션 방식은 중앙의 오케스트레이터가 A -> B -> C -> D로 진행된다는 사실을 알고 있으며 그에 따라 오케스트레이터가 중심이 되어 처리가 진행된다.

![](https://learn.microsoft.com/ko-kr/azure/architecture/reference-architectures/saga/images/orchestrator.png)

그래서 D에서 커밋하지 못하는 상황이 발생하면 D는 오케스트레이터에게 사실을 알린다. 그러면 오케스트레이터가 A, B, C에게 보상 트랜잭션을 실행하라고 알리고 A, B, C는 보상 트랜잭션을 실행한다. 오케스트레이터가 진행 과정을 알고 있다는 게 장점이 되기도 하지만 무언가를 안다는 것은 커플링이 발생한다는 말이기도 하다. 

https://learn.microsoft.com/ko-kr/azure/architecture/reference-architectures/saga/saga#orchestration 에 장단점이 잘 정리돼 있다.

오케스트레이터가 A -> B -> C -> D로 진행되는 흐름을 알고, 다음 단계를 수행할 대상을 명시적으로 집어서 다음 명령(Command)을 수행시키는 방식으로 진행하므로, [마이크로서비스 아키텍처 사이트](https://microservices.io/patterns/data/saga.html)에서는 아래와 같은 오케스트레이션 방식 예제 설명 그림에 Command라는 용어를 사용했다.

![](https://microservices.io/i/sagas/Create_Order_Saga_Orchestration.png)

위 그림에 Message Broker가 나와서 오케스트레이션 방식에서 Message Broker가 필수인 것처럼 보이는데, 어차피 오케스트레이터라는 중앙의 알려진 존재를 통해 진행되므로 오케스트레이터와 A, B, C, D는 Message Broker 없이 직접적인 요청/응답 방식으로 협업할 수도 있다.


## Process Manager Pattern

개인적으로 이 시점에 프로세스 매니저 패턴이 왜 나오는지 모르겠다.

프로세스 매니저 패턴은 일단 https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html 여기에 나오는데, 한 마디로 정의하면 그냥 여러 단계가 있을 때 중앙의 프로세스 매니저가 흐름을 관장(분기 처리, 정상/실패 처리 등)하게 하는 패턴이다.

일부러 아무것도 굵은 글씨로 표시하지 않았다. 개인적으로 이 패턴은 패턴이라고 불러야 할 정도로 특별한 무언가가 없기 때문이다. 여러 단계가 있다면 그 단계를 관장하는 무언가를 두는 것은 너무 자연스러우며 그 무언가가 바로 프로세스 매니저다.

![](https://www.enterpriseintegrationpatterns.com/img/ProcessManager.gif)

오케스트레이션 방식의 사가에 있는 오케스트레이터와 프로세스 매니저 패턴에 있는 프로세스 매니저는 여러 단계로 이루어진 프로세스를 조율/관리한다는 관점에서 굉장히 유사하다. 프로세스 매니저가 조금 더 일반적인 이름이라고 생각한다면 오케스트레이터 역시 프로세스 매니저라고 부를 수도 있다. 다만 오케스트레이터는 데이터 일관성 보장에 더 중심을 둔다는 차이가 있다.


# 마무리

>- Transactional Outbox Pattern과 Saga Pattern은 일관성 보장에 사용한다는 공통점이 있다.
>- Transactional Outbox Pattern은 하나의 Bounded Context 안에서 메시지 브로커로 메시지를 발행할 때 필요한 일관성을 보장해주며, 여러 Bounded Context 사이의 데이터 일관성을 보장해주지는 못한다.
>- Saga Pattern은 보상 트랜잭션을 사용해서 여러 Bounded Context 사이의 데이터 일관성을 결과적 일관성(eventual consistency) 방식으로 보장해주며, 
>    - 결과적 일관성 보장 과장 중에 메시지 브로커를 사용한다면 Bounded Context마다 Transactional Outbox Pattern을 사용해서 각 Bounded Context의 메시지 발행 일관성을 보장할 수 있다.
>- Process Manager는 데이터 일관성 보장과는 크게 관련이 없으며 여러 단계를 가진 프로세스를 관리한다는 관점에서 Orchestration 방식의 Saga에 사용되는 Orchestrator가 일종의 Process Manager라고 볼 수 있다.
>    - Orchestrator는 여러 단계를 가진 프로세스 관리도 하면서 데이터 일관성 보장에 더 중점을 둔다.
