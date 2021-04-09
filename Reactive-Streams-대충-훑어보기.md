# Reactive Streams 대충 훑어보기

1 req == 1 therad인 서블릿 방식의 한계를 뛰어넘기 위해 Spring에서 WebFlux를 내놨다.  
Spring WebFlux는 내부적으로 Spring Reactor를 사용하는데, Spring Reactor는 Reactive Streams 구현체다.  
Reactive Streams는 다음과 같이 간단 명료하게 정의돼 있다.

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

Reactive Streams를 사용해서 비동기로 데이터를 조회하는 시나리오를 기준으로 작성했으며,  
화살표는 위는 메서드 이름, 화살표 아래에 괄호로 표시된 건 메서드 인자, 괄호 없이 표시된 건 반환값이다.

Publisher, Subscriber, Subscription 인터페이스가 가지고 있는 모든 메서드가 표시돼 있다.

![Imgur](https://i.imgur.com/BzOrtx8.png)

### 데이터 핸들러 로직 정의 및 Subscriber 생성

데이터를 요청하는 Client는 데이터를 받아서 어떻게 처리할지, 받는 과정에서 오류를 전달 받으면 어떻게 처리할지, 데이터를 모두 받은 후에 어떤 일을 할지 정해야 할 책임을 가지고 있다. 그런 책임을 각각 `nextConsumer`, `errorConsumer`, `completeRunnable`로 정의해서 이를 바탕으로 `subscriber`를 생성한다.

### Data Provider에 데이터 요청 및 Publisher 생성

그리고 DataProvider에게 데이터를 요청한다. DataProvider는 특정 클래스 이름은 아니고 클라이언트로부터 호출을 받으면 데이터 저장소와 연동해서 실제 데이터를 반환하는 책임이 있는 객체를 의미한다고 보면 된다. 예를 들면 ReactiveMongoOperations(ReactiveMongoTemplate)이나 ReactiveMongoRepository라고 생각하면 된다. 이 DataProvider는 **나중에 데이터를 제공할 수 있도록 콜백을 생성하고 이를 `publisher`를 생성하면서 주입**해준다. 이 부분 자세한 과정은 맨 아래에서 구경할 수 있다.

### 구독

클라이언트는 DataProvider로부터 `publisher`를 반환 받은 후에 `publisher.subscribe(subscriber)`를 호출한다. 리액티브 스트림에서 절대 잊어서는 안 될 가장 중요한 특징 중 하나가 바로 **구독하기 전에는 아무 일도 일어나지 않는다**는 점이다. 즉 앞에서 아무리 `nextConsumer`, `errorConsumer`, `completeRunnable`를 모두 정의하고 DataProvider를 호출해서 데이터를 가져오는 로직을 구현해뒀다 하더라도, `publisher.subscribe(subscriber)`를 호출하지 않으면 앞서 만든 모든 것들은 전혀 호출되지 않는다. 더 정확하게 말하면 **데이터를 가져오는 로직은 아직 콜백에 담겨 있을 뿐이고, 구독하기 전에는 콜백이 실행되지 않는다.**(엄밀하게는 Hot Publisher인 경우 구독과 상관 없이 데이터를 뿜어내지만 앞서 얘기했듯이 Hot은 너무 뜨거워서 생략)

### Subscription 생성

`publisher.subscribe(subscriber)`가 호출되면 `publisher`는 파라미터로 전달받은 `subscriber`와 자신이 생성될 때 주입받은 `dataCallback`를 바탕으로 `subscription`을 생성하고, `subscriber.onSubscribe(subscription)`를 호출해서 `subscription`을 `subscriber`에게 전달해준다.

### Subscription에 데이터 요청 

`subscriber`는 `onSubscribe(subscription)`을 통해 `subscription`을 전달받으면 `subscription.request(numOfData)`를 호출해서 데이터를 요청한다. 자신이 소화할 수 있을 만큼의 데이터만 요청할 수 있으므로 back pressure 개념이 이 지점에서 발동한다. 그리고 실제 데이터 접근도 이 지점에서 이루어진다.

### 실제 데이터 접근 및 onNext/onError/onComplete 호출

`subscription`은 자신이 생성될 때 주입 받은 콜백을 호출해서 `numOfData`만큼만 데이터를 가져오고 `subscriber.onNext(data)`를 반복 호출해서 `subscriber`에게 데이터를 전달한다. 이 과정에서 오류가 발생하면 `subscriber.onError(throwable)`로 오류를 `subscriber`에게 전달하고, 데이터 전달이 정상적으로 완료되면 `subscriber.onComplete()`를 호출하며 협력이 끝난다.


사실 여기까지만 알면 리액티브 스트림의 협력 구조를 이해하는 데는 충분하다고 생각한다.  
물론 그렇다고 리액티르 스트림을 사용하는 데 충분하다는 얘기는 아니다. 실무적으로는 map, flatMap, concatMap, zip 등등 엄\~\~청나게 많은 리액티브 연산자들 예쁘디 예쁜 마블 다이어그램 보면서 다 공부해야지\~\~ ㄷㄷㄷ

암튼 원래는 이 정도만 알고 넘어가려고 했는데.. 이 정도 알아보고 나니.. 또 다른 궁금증이 파생되어..  

하지만 아래에 이어지는 건 굳이 안 봐도 된다. 그냥 개인적인 주절거림일 뿐이다.


## 비동기는 대체 어디에?

리액티브 스트림이 비동기 스트림 처리 표준을 제공하는 킹왕짱 계획이라고 했는데 비동기 처리는 어디에 있는 거임?

사실 리액티브 스트림이 비동기 스트림 처리 표준 제공 어쩌구라고는 하지만 4가지 인터페이스를 보면 비동기 관련 내용은 전혀 없다. 결국 리액티브 스트림은 비동기 처리 표준을 지향하긴 하지만 그렇다고 비동기를 강제하는 것도 아니다.

비동기 처리는 구현에 달려있을 뿐이다.

Spring Reactor에서는 reactor.core.scheduler.Scheduler 인터페이스에 비동기 처리 관련 규약이 담겨 있다. reactor.core.publisher 패키지에서 Scheduler가 사용되는 곳을 검색하면 대략 감이 올 것이다.

[JDK 9 예제](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html)에 보면 Subscription이 Subscriber의 onNext()를 호출할 때 Executor를 이용해서 비동기로 호출하고 있다.


## 깊은 뜻은 모르겠다..

비동기는 어떤 과정을 거치더라도 결국에는 `new Thread(callback).start()`를 거쳐서 실현된다고 봐도 크게 틀리지 않는다. 다양한 비동기 처리 라이브러리는 결국 이 부분을 추상화해서 감출 뿐이다. 감추긴 감추는데 리액티브 스트림에서 이런 방식으로 감추도록 설계한 깊은 뜻까지는 애석하지만 잘 모르겠다.. 라기보다는 솔직히 마음에 들지 않는다.

### 이름 적절한가?

이런 방식이라고 칭한 것 중에는 다음과 같이 좀 어색해 보이는 이름이나 호출 구조도 포함된다. 대표적인 게 `Publisher.subscribe(Subscriber)`다. 퍼블리셔라면 뭔가 데이터를 발행하고 뿜어내야 할 것 같은데 정작 가진 메서드 이름이나 파라미터는 영 다르다.

예전에도 이 부분이 마음에 안 들어서 https://www.facebook.com/hanmomhanda/posts/10214512128821140 여기에 끄적여둔 게 있다. 영문법과 너무 이질적으로 다르다는 점이 불편함을 느낀 시작점이긴 하지만 그것 뿐만은 아니다. 두 가지 관점에서 불편하다.

먼저 `Publisher.subscribe(Subscriber)`가 무슨 동작을 하는지 알아보자. JDK API DOC에도 [`Publisher.subscribe(Subscriber)`에 대한 설명](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.Publisher.html#subscribe-java.util.concurrent.Flow.Subscriber-)은 다음과 같은 간단하고 명료한 문장으로 시작한다.

>Adds the given Subscriber if possible.

즉 실제 하는 동작은 Subscriber를 추가하는 건데 subscribe 라는 엉뚱한 이름이 사용되고 있다. 자기 자신에게 추가하는 것이므로 받아들인다 라고도 볼 수 있으므로, `Publisher.receive(Subscriber)`라고 했으면 실제 동작에 부합하므로 훨씬 직관적이었을 것 같다.

만약 `receive`에 구독 같은 특별한 의미가 결여돼서 의도를 잘 드러내지 못 하는 걸로 보이므로 반드시 `subscribe`라는 이름을 쓰고 싶다면 설계도 바꾸는 게 좋다고 생각한다.  
`Publisher.subscribe(subscriber)`를 통해 subscriber를 추가하는(받아들이는) 이유는, 나중에 Subscription을 생성하고 이를 Subscriber에 전달해주고 Subscriber가 `subscription.request(n)`을 호출하는 데 필요하기 때문이다.  
따라서 `Publisher.subscribe(subscriber)` 대신에 `Subscriber.subscribe(Publihser)`를 통해 Publisher에 Subscriber를 전달해주고, Publisher가 이를 받아서 콜백과 함께 Subscription을 생성하고 이하는 동일하게 가져 갔어도 됐을 것 같다.

아니면 현재 상태에서 아싸리 `Publisher.subscribe(Subscriber)`라는 이름만 그냥 `Publisher.publishTo(Subscriber)`라고 바꿨다면 실제 동작과는 맞지 않는 이름이라 하더라도 API 사용자 입장에서는 훨씬 나았을 것 같다. '구독을 하지 않으면 아무 일도 발생하지 않는다'가 '발행자에게 발행을 요청하지 않으면 아무 일도 발생하지 않는다'로 바뀔 뿐이다. 전혀 어색하지 않을 뿐더러 실질에 오히려 더 잘 부합하는 것 같다.

불편했던 또 한 가지 이유는 onSubscribe, onNext, onError, onComplete 메서드의 컨텍스트인 Subscriber는 주어의 역할을 하는데, Subscriber와 대칭 관계라고 할 수 있는 Publisher는 subscribe 메서드의 주어도 아니고 목적어도 아닌 것 같으니 대칭성이 깨져서 직관성이 떨어지기 때문이다.

그런데 위 페북 글 댓글들을 보면 나를 제외한 나머지 모두는 `Publisher.subscribe(Subscriber)` 이게 어색하지도 해석하는 데 불편하지도 않다는 반응이라서 신기하기도 했다.

### cancel은?

실제 구독을 중단하는 일을 수행하는 책임은 현재 Subscription에 있다. 그리고 `Subscription.cancel()`를 누가 호출하는지 구현체를 살펴보니 본 것 중에서는 모두 Subscriber가 호출하고 있다. 하지만 현재 설계 상으로는 Subscriber에 cancel 관련 공개 메서드가 없으므로, cancel 여부를 결정하는 클라이언트가 cancel을 유발할 수 있는 방법이 없다.

그럼 도대체 cancel이 어떻게 실현될 수 있는 걸까? 구현체인 Reactor를 살펴보니, Reactive Streams 에서는 `Publisher.subscribe()`는 리턴 값이 없지만 리액터의 Publisher 구현체인 Flux에는 Disposable을 반환하는 `subscribe()` 메서드가 있다. 이 Disposable에 `dispose()` 메서드가 있어서 이를 통해 `Subscription.cancel()`을 호출할 수 있다. 그런데 Disposable도 마찬가지로 Reactive Streams에는 없고 구현체인 Reactor에서 만들어 사용하는 인터페이스다.

요는 Reactive Streams만으로는 cancel 여부를 결정하는 클라이언트가 실제 cancel 할 수 있는 방법이 없다.

취소라는 게 구독 취소일 수도 있고 발행 취소일 수도 있으니, cancel 을 Publisher 쪽에 둬도 됐을 것이다. 현재 Publisher에는 내가 보기에는 영 어색한 `subscribe()` 메서드 하나만 달랑 있는데, Publisher에 `cancel()`도 넣어서 클라이언트가 호출할 수 있게 해줬다면 적절했을 것 같다.


## 리액티브 스트림의 가치

리액티브 스트림을 활용한 프로그래밍은 여러모로 진입 장벽이 높다. 그런데 그걸 꼭 넘어서 사용해야할 정도로 가치가 있을까?

비동기 처리라면 C#, JavaScript, Rust 등에는 async/await, Kotlin에는 coroutine 처럼 더 진입 장벽이 낮은 API가 제공되고 있다. 그리고 자바에도 정확히 언제가 될지는 모르지만 Fiber(Project Loom)가 도입될 예정이라고 하니, 비동기 처리라는 관점에서 리액티브가 계속 존속할 수 있을지 솔직히 의문이다.

그러니 깊은 뜻을 굳이 알아야만 할 것 같지는 않다. 그저 back pressure를 적용할 수 있어야 하고, OnNext, onError, onComplete 와 같이 이벤트 핸들링 방식으로 처리하는 API를 제공하려다보니 이런 설계가 나왔겠지 정도로 털고 가자(아 훈훈해..).


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

