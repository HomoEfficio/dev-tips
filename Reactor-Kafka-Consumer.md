# Reactor Kafka Consumer

Spring Reactor를 사용하는 환경에서 카프카와 연동한다면 Reactor Kafka를 사용해야 한다.

이 글에서는 Consumer쪽에 대해서만 다룬다.

그냥 Spring Kafka 에서는 `@KafkaListener`를 사용해서 간편하게 Consumer를 구현할 수 있지만, Reactor Kafka에서는 그렇지 않으며 Reactor를 사용하기 때문에 주의해야할 점이 많은데 하나하나 살펴보자.


## 기본 사용법

Reactor Kafka에서는 다른 방법도 있을 수 있겠지만 다음과 같이 `@EventListener(ApplicationStartedEvent::class)` 방식을 사용해서 스프링 부트 애플리케이션 시작이 완료된 후에 카프카 토픽 consume을 시작하게 만들 수 있다.

```kotlin
@Component
class XxxListener(
    private val kafkaConsumerTemplate: ReactiveKafkaConsumerTemplate<String, Xxx>,
) {
    @EventListener(ApplicationStartedEvent::class)
    fun yyyTopicConsumer() {
        kafkaConsumerTemplate.receive...()

```

## 동작 메커니즘

ReactiveKafkaConsumerTemplate는 내부 private 변수로 KafkaReceiver를 가지고 있고,  
KafkaReceiver는 다음과 같이 Kafka 메시지를 가져올 수 있는 몇 가지 메서드를 정의하고 있다.

```java
Flux<ReceiverRecord<K, V>> receive();

Flux<Flux<ConsumerRecord<K, V>>> receiveAutoAck();

Flux<ConsumerRecord<K, V>> receiveAtmostOnce();

Flux<Flux<ConsumerRecord<K, V>>> receiveExactlyOnce(TransactionManager transactionManager);
```

메서드 이름에서 알 수 있듯이 동작 방식에 차이가 있고, 반환 타입도 조금씩 다르지만, **모두 Flux를 반환한다는 공통점**이 있다. 결국 `kafkaConsumerTemplate.receive...()`를 호출하면 ReactiveKafkaConsumerTemplate는 내부에 있는 KafkaReceiver를 통해 Kafka 토픽 파티션을 구독하고, 파티션에 있는 메시지를 Flux를 통해 받아온다.

더 구체적으로는 KafkaReceiver의 구현체인 DefaultKafkaReceiver가 생성될 때 주입받은 ConsumerFactory를 이용해서 만든 KafkaConsumer의 poll() 메서드를 주기적으로 호출하고, 호출할 때마다 내부적으로 Fetcher.fetchRecords()를 호출해서 파티션별 레코드들을 `Map<TopicPartition, List<ConsumerRecord<K, V>>`에 담아 받아온다는 점은 동일하다.

조금 더 깊이 들어가보면 InitEvent, PollEvent, CommitEvent, CloseEvent, CustomEvent를 Kafka로 보내는 통로 역할을 하는 Flux와 PollEvent에 의해 Kafka로부터 레코드를 받아오는 통로 역할을 하는 Flux가 별도로 존재하는데, 위의 네 가지 메서드의 반환 타입 가장 바깥 쪽에 있는 Flux가 바로 레코드를 받아오는 통로 역할을 하는 Flux다.

참고로 KafkaConsumer는 Apache Kafka에서 제공되는 클래스로서 topic, partition, ofsset, headers, key, value 등 정보를 포함하는 ConsumerRecord를 다루고, KafkaReceiver는 Reactor Kafka에서 제공되는 클래스로서 내부 변수로 ConsumerFactory를 가지고 있다.
ReceiverRecord는 ConsumerRecord를 상속받은 클래스이고, 레코드별로 명시적으로 acknowledge를 할 수 있는 ReceiverOffset을 내부변수로 가지고 있다.


## 라이프사이클

Flux를 통해 메시지를 받고 처리하던 중 예외가 발생해서 Flux가 종료된다면 어떻게 될까?

컨수머는 더 이상 파티션에서 [메시지를 받을 수 없고](https://projectreactor.io/docs/kafka/release/api/reactor/kafka/receiver/KafkaReceiver.html#receive--) 결국 해당 컨수머는 컨수머 그룹에서 이탈하고 Kafka에서는 리밸런싱이 발생한다. 그럼에도 불구하고 위 `XxxListener` 인스턴스 자체는 스프링 빈으로서 계속 존재한다. 컨수머라는 역할을 제대로 수행하지 못하면서도 그대로 남아있는다.

따라서 예외가 발생하더라도 적절한 예외 처리를 통해 Flux가 종료되지 않도록 하든가, 아니면 Flux 종료 시 새로운 컨수머를 붙여줘야 기존의 성능을 유지할 수 있다.


## 컨숨 방식

앞서 살펴본 것처럼 4가지 방식이 있다(1.3.21에서 `receiveBatch()`가 추가됐다). 이중 `receiveAtMostOnce()`는 적절히 사용하지 않는다면 메시지 처리가 누락될 위험이 있고, `receiveExactlyOnce()`는 성능에 악영향을 줄 위험이 있어 사용에 주의가 필요하다. 그래서 실무적으로는 At-Most-Once나 Exactly-Once 대신에 At-Least-Once와 멱등 처리를 주로 사용하므로 여기에서도 `receiveAutoAck()`와 `receive()`만 다룬다.


### receiveAutoAck()

이름에서 알 수 있듯이 자동으로 acknowledge를 수행한다. acknowledge 대상은 무엇일까? `Consumer.poll(long)`을 통해 가져오는 한 묶음(batch)의 레코드들이 acknowledge 대상이다.

레코드들을 하나의 Flux로 묶어서 가져오므로 `receiveAutoAck()`는 `Flux<Flux<ConsumerRecord<K, V>>>`를 반환한다. 가장 바깥에 있는 Flux는 파티션과 연결된 통로 역할을 하는 Flux이고 그 안에 있는 `Flux<ConsumerRecord<K, V>`는 한 묶음의 레코드들이다. 가장 바깥에서 연결 통로를 하는 Flux를 통해 여러 개의 `Flux<ConsumerRecord<K, V>>`를 받아서 처리에 성공하면 acknowledge를 자동으로 수행한다.

ReactiveKafkaConsumerTemplate를 사용한다면 ReactiveKafkaConsumerTemplate.receiveAutoAck()를 호출하는데, 이 메서드는 다음과 같이 정의돼 있다.

```java
    public Flux<ConsumerRecord<K, V>> receiveAutoAck() {
        return this.kafkaReceiver.receiveAutoAck().flatMap(Function.identity());
    }
```

즉 KafkaReceiver가 반환하는 `Flux<Flux<ConsumerRecord<K, V>>>`에서 `flatMap(Function.identity())`을 통해 Flux를 한 꺼풀 벗겨내고 `Flux<ConsumerRecord<K, V>>`를 반환한다. 

### receive()

ConsumerRecord를 ReceiverRecord로 바꿔주는 역할



### receiveAtMostOnce()
### receiveExactlyOnce()
### receiveBatch() since 1.3.21

- 수동 commit

## Flux 처리 방식

### flatMap

- 메시지 발행 순서와 commit 순서가 꼬일 수 있어서 at-least-once 방식에서는 문제 발생 위험
  - 메시지1, 2, 3이 순서대로 들어오면, 병렬로 처리되면서 메시지3이 가장 먼저 처리 완료되어 commit 되고, 메시지1,2처리가 완료되기 전에 컨수머가 다운되면, 나중에 컨수머가 다시 카프카에 붙었을 때 메시지1,2는 다시 가져오지 못하므로 결과적으로 메시지1,2는 처리되지 않게 되어 at-least-once가 사실상 깨지게 된다.

### flatMapSequential

- 메시지1, 2, 3이 순서대로 들어오면, 병렬로 처리되면서 메시지3이 가장 먼저 처리 완료되어도 commit하지 않고 메시지1, 2가 처리 완료될 때까지 기다린 후에 메시지1, 2, 3을 순서대로 downstream으로 보내서 순서대로 commit하므로 at-least-once가 된다.

### concatMap

- 메시지1, 2, 3이 순서대로 들어오면 들어오는 순서대로 처리되므로 메시지2가 메시지 1보다 먼저 처리되는 일이 발생하지 않고 commit 순서도 유지되므로 at-least-once가 보장된다.


## 위치 조정

- 컨수머는 기본적으로 파티션별 last committed offset을 기준으로 consume을 시작하지만 last committed offset을 알 수 없을 때는 ConsumerConfig#AUTO_OFFSET_RESET_CONFIG에 설정된 값(earliest 또는 latest)을 기준으로 consume을 시작한다.
- 하지만 ReceiverOptions를 사용하면 last committed offset 말고 원하는 위치부터 consume하도록 만들 수 있다.

```
ReceiverOptions.create<KEY, VALUE>(properties)
    .addAssignListener { partitions -> 
        partitions.forEach { p -> p.seek(333L) }
    }
    ...
    .subscription(Collections.singleton(topic));
KafkaReceiver.create(receiverOptions).receive().subscribe();
// 또는 ReactiveKafkaConsumerTemplate<KEY, VALUE>(receiverOptions)
```

void seekToBeginning();
void seekToEnd();
void seek(long offset);
void seekToTimestamp(long timestamp);






## 에러 처리



이렇게 하면

- batch 단위로 받아와서 Flux로 반환한다
- Flux 처리 중 에러가 발생하면 컨수머 그룹에서 이탈하고 더 이상 메시지를 처리하지 못하지만 서비스 인스턴스 자체는 그대로 유지되어 겉으로는 티가 안 나게 된다.
- 별다른 조치를 취하지 않으면 해당 컨수머가 연결돼 있던 파티션에 메시지 적체가 발생한다.
  - retryWhen을 통해 일시적 에러(네트워크 에러 등 일시적으로 발생하여 Kafka consumer에 의해 던져진 예외)에 대해 대비할 수 있으며, 기타 애플리케이션에서 발생하는 예외는 알림 등 적절한 처리를 해야 한다.




## 참고

- https://dzone.com/articles/resilient-kafka-consumers-with-reactor-kafka
- https://projectreactor.io/docs/kafka/snapshot/reference/
