title: Reactive Streams with Sequence Diagram
date: 2021-04-14 00:28:04
categories:
  - Technique
tags:
  - Reactive Streams
  - Reactive Streams Sequence Diagram
  - Spring
  - Reactor
  - Sequence Diagram
  - Async
  - Publisher
  - Subscriber
  - Subscription
thumbnailImage: https://i.imgur.com/ZVo09KP.png
coverImage: cover-ReactiveStreams-SequenceDiagram.png
---
# Reactive Streams with Sequence Diagram

1 req == 1 therad인 서블릿 방식의 한계를 뛰어넘기 위해 Spring에서 WebFlux를 내놨다.  
Spring WebFlux는 내부적으로 Spring Reactor를 사용하는데, Spring Reactor는 Reactive Streams 구현체다.  
Reactive Streams는 [홈페이지](https://www.reactive-streams.org/)에 다음과 같이 간단 명료하게 정의돼 있다.

>Reactive Streams is an initiative to provide a standard for asynchronous stream processing with non-blocking back pressure.
>
>리액티브 스트림은 비동기 스트림 처리 표준을 제공하는 킹왕짱 계획이야. 논블로킹 백프레셔도 있다능!

솔직히 뭔 소린지 모르겠다. 이거 안다고 퇴근 시간이 앞당겨질 것 같지는 않은데 몰라도 너무 모르니 한 번 알아보려 한다.

이후 나오는 내용은 [JDK 9 Flow 예제](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html)와 Spring Reactor, Reactive MongoDB에서 구현한 여러 Reactive Streams 구현 내용을 기준으로 작성했으며, Hot Publisher는 배제하고 Cold Publisher만 다룬다.


## 등장 인물

Reactive Streams는 Publisher, Subscriber, Subscription, Processor 달랑 네 개의 인터페이스로 구성돼있다.  
이 중에서 Processor는 자기만의 고유한 메서드는 없고 단순히 Publisher, Subscriber 두 가지 인터페이스를 상속받는다. Processor 없이 나머지 3개만으로도 Reactive Streams 협력을 구성할 수 있으므로 Processor는 등장 인물에서 제외한다.

얘네들이 각각 무슨 동작을 가지고 있고 어떻게 협력해서 클라이언트에게 데이터를 반환하는지 한 눈에 알아보자.


## 시퀀스 다이어그램

검색해보면 몇 가지 나오기는 하는데 내가 이해할 수 있을 만큼 마음에 드는 게 없어서 새로 그렸다(개고생 ㅠㅜ).  

Publisher, Subscriber, Subscription 인터페이스가 가지고 있는 모든 메서드가 표시돼 있다. publisher 의 `map, flatMap, zip, ...`는 Reactive Streams 명세에 있는 메서드가 아니라 구현체인 Reactor의 Flux에 있는 메서드인데 설명의 편의를 위해 추가했다.

Reactive Streams를 사용해서 비동기로 데이터를 조회하는 시나리오를 기준으로 작성했으며,  
실선 화살표는 메서드 호출, 화살표 위는 메서드 이름, 화살표 아래에 괄호로 표시된 건 메서드 인자이며, 점선 화살표는 반환이고 화살표 아래 괄호 없이 표시된 건 반환값이다.

![Imgur](https://i.imgur.com/kbp9BGI.png)

### 데이터 핸들러 로직 정의 및 Subscriber 생성

데이터를 요청하는 Client는 데이터를 받아서 어떻게 처리할지, 받는 과정에서 오류를 전달 받으면 어떻게 처리할지, 데이터를 모두 받은 후에 어떤 일을 할지 정해야 할 책임을 가지고 있다. 그런 책임을 각각 `nextConsumer`, `errorConsumer`, `completeRunnable`로 정의해서 이를 바탕으로 `subscriber`를 생성한다.

설명의 편의를 위해 `subscribe` 생성을 가장 먼저 표시했는데 그림을 보면 알 수 있겠지만 반드시 가장 먼저 수행할 필요는 없다. `publisher.subscribe(subscriber)`를 호출하기 직전에 `subscriber`를 생성해도 된다.

### Data Provider에 데이터 요청 및 Publisher 생성

클라이언트는 DataProvider에게 데이터를 요청한다. DataProvider는 특정 클래스 이름은 아니고 클라이언트로부터 호출을 받으면 데이터 저장소와 연동해서 실제 데이터를 반환하는 책임이 있는 객체를 의미한다고 보면 된다. 예를 들면 ReactiveMongoOperations(ReactiveMongoTemplate)이나 ReactiveMongoRepository라고 생각하면 된다. 이 DataProvider는 **나중에 데이터를 제공할 수 있도록 콜백을 생성하고 이를 `publisher`를 생성하면서 주입**해준다. 이 부분 자세한 과정은 맨 아래에서 구경할 수 있다. DataProvider는 생성한 `publisher`를 클라이언트에게 반환한다.

### 구독

클라이언트는 DataProvider로부터 `publisher`를 반환 받은 후에는, 나중에 `publisher`가 발행할 데이터를 받아서 비즈니스 요구에 맞게 가공하는 로직을 추가한다. `map`, `flatMap`, `zip` 등 여러 리액티브 연산자가 이 때 사용된다.

클라이언트는 데이터 가공 로직 추가를 마친 후에`publisher.subscribe(subscriber)`를 호출한다. 리액티브 스트림에서 절대 잊어서는 안 될 가장 중요한 특징 중 하나가 바로 **구독하기 전에는 아무 일도 일어나지 않는다**는 점이다. 즉 앞에서 아무리 `nextConsumer`, `errorConsumer`, `completeRunnable`를 모두 정의하고, DataProvider를 호출해서 데이터를 가져오고 가공하는 로직을 구현해뒀다 하더라도, `publisher.subscribe(subscriber)`를 호출하지 않으면 앞서 만든 모든 것들은 전혀 실행되지 않는다. 더 정확하게 말하면 **데이터를 가져오는 로직은 아직 콜백에 담겨 있을 뿐이고, 구독하기 전에는 콜백이 실행되지 않는다.**(엄밀하게는 Hot Publisher인 경우 구독과 상관 없이 데이터를 뿜어내지만 앞서 얘기했듯이 Hot은 너무 뜨거워서 생략)

### Subscription 생성

`publisher.subscribe(subscriber)`가 호출되면 `publisher`는 인자로 전달받은 `subscriber`와 자신이 생성될 때 주입받은 `dataCallback`를 바탕으로 `subscription`을 생성하고, `subscriber.onSubscribe(subscription)`를 호출해서 `subscription`을 `subscriber`에게 전달해준다.

### Subscription에 데이터 요청 

`subscriber`는 `onSubscribe(subscription)`을 통해 `subscription`을 전달받으면 `subscription.request(numOfData)`를 호출해서 데이터를 요청한다. 자신이 소화할 수 있을 만큼의 데이터만 요청할 수 있으므로 back pressure 개념이 이 지점에서 발동한다. 그리고 실제 데이터 접근도 이 시점에서 이루어진다.

### 실제 데이터 접근 및 onNext/onError/onComplete 호출

`subscription`은 자신이 생성될 때 주입 받은 콜백을 호출해서 `numOfData`만큼만 데이터를 가져오고 `subscriber.onNext(data)`를 반복 호출해서 `subscriber`에게 데이터를 전달한다. 이 과정에서 오류가 발생하면 `subscriber.onError(throwable)`로 오류를 `subscriber`에게 전달하고, 데이터 전달이 정상적으로 완료되면 `subscriber.onComplete()`를 호출하며 협력이 끝난다.


## 비동기는 대체 어디에?

리액티브 스트림이 백프레셔와 함께 비동기 스트림 처리 표준을 제공하는 킹왕짱 계획이라고 했는데, 이 협력 구조 상에서 비동기 처리는 어디에 있는 걸까?

사실 리액티브 스트림이 비동기 스트림 처리 표준 제공 어쩌구라고는 하지만 4가지 인터페이스를 보면 비동기 관련 내용은 전혀 없다. 다시 말해 비동기 처리 없이 동기 처리만 사용하더라도 스트림을 리액티브 방식으로 처리하는 것이 가능하다. 결국 **리액티브 스트림은 비동기 처리 표준을 지향하긴 하지만 그렇다고 비동기를 강제하는 것도 아니다.** 따라서 비동기 처리는 실질적으로는 구현에 달려 있다.

Spring Reactor의 비동기 처리 관련 규약은 `reactor.core.scheduler.Scheduler` 인터페이스에 담겨 있다. `reactor.core.publisher` 패키지에서 Scheduler가 사용되는 곳을 검색하면 어떻게 비동기 처리를 하는지 대략 감을 잡을 수 있을 것이다.

또 다른 예로 [JDK 9 예제](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html)에 보면 `Subscription`이 `Subscriber`의 `onNext()`를 호출할 때 `Executor`를 이용해서 비동기로 호출하고 있다.


## 마무리

>Reactive Streams는 논블로킹, 백프레셔를 포함해서 스트림을 비동기 방식으로 처리할 수 있는 표준 API다.
>
>Publisher, Subscriber, Subscription 이 서로 협력하면서 스트림을 리액티브 방식으로 처리한다.  
>자세한 협력 구조는 글보다는 [시퀀스 다이어그램](https://i.imgur.com/kbp9BGI.png)을 참고하면 더 쉽게 이해할 수 있다.


---

# 번외편

사실 여기까지만 알면 리액티브 스트림의 협력 구조를 이해하는 데는 충분하다고 생각한다.  
물론 그렇다고 리액티브 스트림을 사용하는 데 충분하다는 얘기는 아니다. 실무적으로는 `map`, `flatMap`, `concatMap`, `zip` 등등 엄\~\~청나게 많은 리액티브 연산자들 예쁘디 예쁜 마블 다이어그램 보면서 다 공부해야 된다능\~\~ ㄷㄷㄷ

암튼 원래는 이 정도만 알고 넘어가려고 했는데.. 이 정도 알아보고 나니 또 다른 궁금증이 파생되어..  
아래에 이어지는 건 굳이 안 봐도 된다. 그냥 개인적인 주절거림일 뿐이다.


## 이런 이름 적절한가?

`Publisher`를 보자. 퍼블리셔라면 뭔가 데이터를 발행하고 뿜어내야 할 것 같은데 정작 가진 메서드 이름이나 파라미터는 영 다르다. 달랑 하나 있는 메서드는 다음과 같다.

>`Publisher.subscribe(Subscriber)`
>
>발행자가 구독한다 구독자를. 읭? 발행자가 구독자를 구독한다고? 뭥미?

일단 구독은 구독자의 행위인데 구독자가 아니라 엉뚱하게 발행자가 주어로 나와있다. 사실 이렇게 행위의 주어와 메서드가 소속된 객체가 다르게 돼 있는 API는 많다. 모든 getter 메서드는 get이라는 행위의 주어와 getXXX라는 메서드가 소속된 객체가 다르다. 멀리 갈 것 없이 위 시퀀스 다이어그램에 나오는 `subscription.request(numOfData)`도 마찬가지다. request 행위의 주어는 `subscriber`지만 `request()` 메서드는 `subscription`에 소속돼 있다. 그래도 이 코드를 읽는 데는 전혀 불편함이 없다. 결국 **행위의 주어와 메서드가 소속된 객체가 다르다는 것만으로는 불편함을 느끼지 않는다.**

하지만 **`Publisher.subscribe(Subscriber)`는 단순히 행위의 주어와 메서드가 소속된 객체가 불일치하는 데 그치지 않고, 발행자/구독자라는 대칭 관계에 있는 애들이 구독한다(subscribe)라는 행위(동사)를 중심으로 주어, 목적어가 뒤바뀌어 있다.** 그래서 직관적으로 자연스럽게 협력 구조를 이해하는 데 큰 걸림돌이 된다.

단순히 문장 해석하듯 자연스럽게 이해되지 않는 것뿐만 아니라 불편한 이유는 몇 가지 더 있다.

먼저 이 `subscribe`라는 이름은 실제 동작과도 부합하지 않는다.

JDK API 문서에 나오는 [`Publisher.subscribe(Subscriber)`에 대한 설명](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.Publisher.html#subscribe-java.util.concurrent.Flow.Subscriber-)은 다음과 같은 간단하고 명료한 문장으로 시작한다.

>Adds the given Subscriber if possible.

즉 **실제 하는 동작은 Subscriber를 add 하는 건데 subscribe 라는 엉뚱한 이름이 사용되고 있다.**

게다가 Reactive는 Push 기반 데이터 처리 패턴이라고 할 수 있는데, 데이터 처리 실행을 유발하는 핵심 동작을 구독자 입장의 동작인 `subscribe`라고 이름지어버리니까 Publisher가 Push하는 패턴이 아니라 Subscriber가 Pull하는 패턴처럼 느껴진다. 어쩌면 Push를 퇴색시키는 이 부분이 Reactive를 이해하고 사용하는 데 있어 가장 나쁜 점일 수도 있겠다. 이쯤되면 불편함을 넘어 해로움이라고 할 수도 있겠다.

불편했던 또 한 가지 이유는 onSubscribe, onNext, onError, onComplete 메서드의 컨텍스트인 Subscriber는 주어의 역할을 하는데, Subscriber와 대칭 관계라고 할 수 있는 Publisher는 subscribe 메서드의 주어도 아니고 목적어도 아니니 대칭성이 깨져서 직관성이 떨어지기 때문이다.

암튼 내가 보기엔 불편함을 넘어 해로워보이기까지 하는 `subscribe(Subscriber)`가 사용된 이유는 아마도 답정너처럼 Reactive Streams 협력 구조에서 `subscribe`라는 용어를 중심으로 내세우고 싶었던 것이 아니었을까 짐작해본다.

예전에도 이 부분이 마음에 안 들어서 https://www.facebook.com/hanmomhanda/posts/10214512128821140 여기에 끄적여둔 게 있다. 댓글들을 보면 나를 제외한 나머지 모두는 `Publisher.subscribe(Subscriber)` 이게 어색하지도 해석하는 데 불편하지도 않다는 반응이라서 신기하기도 했다.

`Publisher.subscribe(Subscriber)`가 불편하다면, Publisher/Subscriber와 비슷하게 Producer/Consumer 관계가 사용되는 `Stream.forEach(Consumer)`도 불편해야 하는 거 아니냐는 날카로운 의견도 있었는데, 일단 Stream이 역할은 Producer 이긴 하지만 API 상 공식 이름은 어디까지나 Stream이다. Stream.forEach(Consumer)는 스트림 안에서 흐르는 각 원소에 대해 Consumer가 소비한다 외에 다른 해석을 떠올리기 어려울 정도로 직관적이므로 `Publisher.subscribe(Subscriber)`와는 많이 다르다고 생각한다.


## cancel은?

실제 구독을 중단하는 일을 수행하는 책임은 현재 Subscription에 있다. 그리고 `Subscription.cancel()`를 누가 호출하는지 구현체를 살펴보니 본 것 중에서는 모두 Subscriber가 호출하고 있다. 하지만 현재 설계 상으로는 Subscriber에 cancel 관련 공개 메서드가 없으므로, cancel 여부를 결정하는 클라이언트가 cancel을 유발할 수 있는 방법이 없다.

그럼 도대체 cancel이 어떻게 실현될 수 있는 걸까? 구현체인 Reactor를 살펴보니, Reactive Streams 에서는 `Publisher.subscribe()`는 리턴 값이 없지만 리액터의 Publisher 구현체인 Flux에는 Disposable을 반환하는 `subscribe()` 메서드가 있다. 이 Disposable에 `dispose()` 메서드가 있어서 이를 통해 `Subscription.cancel()`을 호출할 수 있다. 그런데 Disposable도 Reactive Streams에는 없고 구현체인 Reactor에서 만들어 사용하는 인터페이스다.

요는 Reactive Streams만으로는 cancel 여부를 결정하는 클라이언트가 cancel 할 수 있는 방법이 없다.


## 개선된 시퀀스 다이어그램

궁시렁대기만 할 게 아니라 개선도 한 번 생각해보자.

`subscribe`라는 용어를 중심에 두고 싶었다면 협력 구조도 좀 바꿔서 `Subscriber.subscribe(Publisher)`로 했으면 어땠을까? 이렇게 하면 클라이언트가 `Subscriber`와만 직접 통신하게 되고, 너무 구독만 강조돼서 Push 기반이라는 Reactive의 특징이 퇴색되는 안 좋은 결과로 이어진다. 게다가 코드도 아래와 같이 못난이가 돼버린다.

```java
subscriber.subscribe(
    Flux.just("a", "b", "c")
        .filter(...)
        .map(...)
        ...
);
```

결국 컨텍스트인 `Publisher`를 `Subscriber`로 바꾸는 건 좋은 대안이 아닌 걸로 보인다. 그렇다면 `Publisher`는 그대로 두고 `subscribe(Subscriber)` 대신 실제 동작에 맞게 `add(Subscriber)`로 하거나, add가 너무 일반적인 동사라 의도를 표현하는 데 부족하다면 `addSubscriber(Subscriber)`로 했다면 훨씬 이해하기 편했을 것 같다. 그런데 이렇게 하면 '구독하지 않으면 아무 일도 일어나지 않는다'라는 문장과 어긋나는 면이 있고, 여전히 계속 구독만 강조되는 것처럼 보인다.

'추가'라는 표면적인 동작과는 맞지 않지만 결국에는 구독자에게 발행한다는 의미에 무게를 둬서 `publishTo(Subscriber)`로 하면 어떨까? 일단 이렇게 하면 '발행자가 구독자에게 발행한다'라고 아주 직관적으로 이해할 수 있게 된다. 게다가 Push 모델이라는 점도 훨씬 극명하게 드러난다. '구독하지 않으면 아무 일도 일어나지 않는다' 대신에 '발행하지 않으면 아무 일도 일어나지 않는다'로 바꿔도 전혀 어색하지 않다.

이름 하나 바꿨을뿐인데 모든 게 좋아진 것 같(은 환각이 든)다. cancel도 Publisher에게 추가하는 것이 좋겠다.

이를 반영해서 개선한 시퀀스 다이어그램은 다음과 같다. 딸기색 둥근 네모 부분만 달라졌다.

![Imgur](https://i.imgur.com/84eEoAS.png)


## 비동기 처리 관점에서 리액티브 스트림의 가치

리액티브 스트림을 활용한 프로그래밍은 여러모로 진입 장벽이 높다. 그런데 그걸 꼭 넘어서 사용해야할 정도로 가치가 있을까?

비동기 처리라면 C#, JavaScript, Rust 등에는 async/await, Kotlin에는 coroutine 처럼 더 진입 장벽이 낮은 API가 제공되고 있다. 그리고 자바에도 정확히 언제가 될지는 모르지만 Fiber(Project Loom)가 도입될 예정이므로 **비동기 처리라는 관점에서 리액티브 스트림이나 ReactiveX가 앞서 예를 든 더 간편한 API들과 견주어 경쟁력을 유지할 수 있을지 솔직히 의문**이다. 한 예로 Kotlin Coroutine과 Reactive Streams 코드를 비교한 자료(https://github.com/HomoEfficio/dev-tips/blob/master/Kotlin-Coroutine-vs-Reactive-Streams(Reactor).md) 를 보면 이런 의문을 가질 법하다는 사실을 실감나게 느낄 수 있을 것이다.

따라서 비동기 처리 관점에서 리액티브 스트림을 아주 깊게 이해해야만 할 것 같지는 않다. 그저 back pressure를 적용할 수 있어야 하고, `onNext`, `onError`, `onComplete` 와 같이 이벤트 핸들링 방식으로 처리하는 API를 제공하려다보니 이런 설계가 나왔겠지 정도로 털고 가자(아 훈훈해..).

물론 꼭 **비동기 처리를 필요로 하지 않는 Push 기반의 데이터 처리 패턴으로서의 존재 가치는 여전히 유효**할 것이다.


## 콜백이 어디에 어떻게 감춰져 있는지 구경하기

그럼 Reactor와 Reactive MongoDB가 추상화해서 감춰든 부분을 한 번 구경해보자. 위에서 아래로 호출이 이어진다고 보고 흐름을 구경하면 된다.

```java
// ReactiveMongoTemplate

    protected <T> Mono<T> doFindOne(String collectionName, Document query, @Nullable Document fields,
            Class<T> entityClass, FindPublisherPreparer preparer) {

        MongoPersistentEntity<?> entity = mappingContext.getPersistentEntity(entityClass);

        QueryContext queryContext = queryOperations
                .createQueryContext(new BasicQuery(query, fields != null ? fields : new Document()));
        Document mappedFields = queryContext.getMappedFields(entity, entityClass, projectionFactory);
        Document mappedQuery = queryContext.getMappedQuery(entity);

        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug(String.format("findOne using query: %s fields: %s for class: %s in collection: %s",
                    serializeToJsonSafely(query), mappedFields, entityClass, collectionName));
        }

        // 여기!!
        return executeFindOneInternal(new FindOneCallback(mappedQuery, mappedFields, preparer),
                new ReadDocumentCallback<>(this.mongoConverter, entityClass, collectionName), collectionName);
    }
    

    private <T> Mono<T> executeFindOneInternal(ReactiveCollectionCallback<Document> collectionCallback,
            DocumentCallback<T> objectCallback, String collectionName) {

        // 여기!!
        return createMono(collectionName,
                collection -> Mono.from(collectionCallback.doInCollection(collection)).flatMap(objectCallback::doWith));
    }


    public <T> Mono<T> createMono(String collectionName, ReactiveCollectionCallback<T> callback) {

        Assert.hasText(collectionName, "Collection name must not be null or empty!");
        Assert.notNull(callback, "ReactiveCollectionCallback must not be null!");

        Mono<MongoCollection<Document>> collectionPublisher = doGetDatabase()
                .map(database -> getAndPrepareCollection(database, collectionName));

                                                                   // 여기!!
        return collectionPublisher.flatMap(collection -> Mono.from(callback.doInCollection(collection)))
                .onErrorMap(translateException());
    }



    private static class FindOneCallback implements ReactiveCollectionCallback<Document> {

        private final Document query;
        private final Optional<Document> fields;
        private final FindPublisherPreparer preparer;

        FindOneCallback(Document query, @Nullable Document fields, FindPublisherPreparer preparer) {
            this.query = query;
            this.fields = Optional.ofNullable(fields);
            this.preparer = preparer;
        }

        @Override
        public Publisher<Document> doInCollection(MongoCollection<Document> collection)
                throws MongoException, DataAccessException {

            if (LOGGER.isDebugEnabled()) {

                LOGGER.debug("findOne using query: {} fields: {} in db.collection: {}", serializeToJsonSafely(query),
                        serializeToJsonSafely(fields.orElseGet(Document::new)), collection.getNamespace().getFullName());
            }

                                                        // 여기!!
            FindPublisher<Document> publisher = preparer.initiateFind(collection, col -> col.find(query, Document.class));

            if (fields.isPresent()) {
                publisher = publisher.projection(fields.get());
            }

            return publisher.limit(1).first();
        }
    }



// FindPublisherPreparer
    default FindPublisher<Document> initiateFind(MongoCollection<Document> collection,
            Function<MongoCollection<Document>, FindPublisher<Document>> find) {

        Assert.notNull(collection, "Collection must not be null!");
        Assert.notNull(find, "Find function must not be null!");

        if (hasReadPreference()) {
            collection = collection.withReadPreference(getReadPreference());
        }
               // 여기!!
        return prepare(find.apply(collection));
    }


// ReactiveMongoTemplate
        public FindPublisher<Document> prepare(FindPublisher<Document> findPublisher) {

            FindPublisher<Document> findPublisherToUse = operations.forType(type) //
                    .getCollation(query) //
                    .map(Collation::toMongoCollation) //
                    .map(findPublisher::collation) //
                    .orElse(findPublisher);

            // findPublisherToUse 에 limit, skip, hint 등 적용한 후에 반환
            // ...

            return findPublisherToUse;
        }


// FindPublisehrImpl
final class FindPublisherImpl<TResult> implements FindPublisher<TResult> {

    private final AsyncFindIterable<TResult> wrapped;

    FindPublisherImpl(final AsyncFindIterable<TResult> wrapped) {
        this.wrapped = notNull("wrapped", wrapped);
    }

    @Override
    public Publisher<TResult> first() {
               // 여기!!
        return Publishers.publish(wrapped::first);
    }


//Publishers
public static <TResult> Publisher<TResult> publish(final Block<SingleResultCallback<TResult>> operation) {
           // 여기!!
    return subscriber -> new SingleResultCallbackSubscription<>(operation, subscriber);
}
```

