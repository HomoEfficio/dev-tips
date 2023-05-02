# Kafka Poison Pill Spring ErrorHandlingDeserializer

카프카 메시지 consume은 대략 다음과 같은 절차로 진행된다.

1. Fetch Serialized Data from Broker(직렬화된 데이터를 브로커로부터 가져와서)
1. Deserialize the fetched serialized data(정해진 타입으로 역직렬화하고)
1. Consume the deserialized data(역직렬화한 데이터를 처리하고)
1. Commit(브로커에게 commit 신호 전송)

## 받아온 메시지 처리에 실패하면..

스프링 카프카를 사용하면서 개발자가 @KafkaListener를 붙여 작성한 리스너 구현부는 3번 과정에 해당한다.  
3번 과정에서 에러가 발생하면 정해진 횟수(기본 10회)만큼 재시도 후 끝까지 실패하면 대략 아래와 같은 에러 로그를 남기면서 그냥 4번 commit 신호를 브로커로 보낸다.  

```
2022-08-27 22:50:06.326 ERROR 64718 --- [ntainer#0-0-C-1] o.s.kafka.listener.DefaultErrorHandler   : Backoff FixedBackOff{interval=0, currentAttempts=10, maxAttempts=9} exhausted for basic-topic-01-1@2

org.springframework.kafka.listener.ListenerExecutionFailedException: Listener method 'public void io.homo_efficio.scratchpad.spring.reactor.kafka.service.KafkaConsumer.listenerBasicTopic01(java.lang.String)' threw exception; nested exception is java.lang.RuntimeException: 4th 메시지는 처리 불가!!!; nested exception is java.lang.RuntimeException: 4th 메시지는 처리 불가!!!
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.decorateException(KafkaMessageListenerContainer.java:2703) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeOnMessage(KafkaMessageListenerContainer.java:2673) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeOnMessage(KafkaMessageListenerContainer.java:2633) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeRecordListener(KafkaMessageListenerContainer.java:2560) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeWithRecords(KafkaMessageListenerContainer.java:2441) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeRecordListener(KafkaMessageListenerContainer.java:2319) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeListener(KafkaMessageListenerContainer.java:1990) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeIfHaveRecords(KafkaMessageListenerContainer.java:1366) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollAndInvoke(KafkaMessageListenerContainer.java:1357) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.run(KafkaMessageListenerContainer.java:1252) ~[spring-kafka-2.8.7.jar:2.8.7]
    at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:264) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run(FutureTask.java) ~[na:na]
    at java.base/java.lang.Thread.run(Thread.java:832) ~[na:na]
    Suppressed: org.springframework.kafka.listener.ListenerExecutionFailedException: Restored Stack Trace
        at org.springframework.kafka.listener.adapter.MessagingMessageListenerAdapter.invokeHandler(MessagingMessageListenerAdapter.java:363) ~[spring-kafka-2.8.7.jar:2.8.7]
        at org.springframework.kafka.listener.adapter.RecordMessagingMessageListenerAdapter.onMessage(RecordMessagingMessageListenerAdapter.java:92) ~[spring-kafka-2.8.7.jar:2.8.7]
        at org.springframework.kafka.listener.adapter.RecordMessagingMessageListenerAdapter.onMessage(RecordMessagingMessageListenerAdapter.java:53) ~[spring-kafka-2.8.7.jar:2.8.7]
        at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeOnMessage(KafkaMessageListenerContainer.java:2653) ~[spring-kafka-2.8.7.jar:2.8.7]
Caused by: java.lang.RuntimeException: 4th 메시지는 처리 불가!!!
    at io.homo_efficio.scratchpad.spring.reactor.kafka.service.KafkaConsumer.listenerBasicTopic01(KafkaConsumer.kt:13) ~[main/:na]
    at jdk.internal.reflect.GeneratedMethodAccessor85.invoke(Unknown Source) ~[na:na]
    at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
    at java.base/java.lang.reflect.Method.invoke(Method.java:564) ~[na:na]
    at org.springframework.messaging.handler.invocation.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:169) ~[spring-messaging-5.3.21.jar:5.3.21]
    at org.springframework.messaging.handler.invocation.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:119) ~[spring-messaging-5.3.21.jar:5.3.21]
    at org.springframework.kafka.listener.adapter.HandlerAdapter.invoke(HandlerAdapter.java:56) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.adapter.MessagingMessageListenerAdapter.invokeHandler(MessagingMessageListenerAdapter.java:347) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.adapter.RecordMessagingMessageListenerAdapter.onMessage(RecordMessagingMessageListenerAdapter.java:92) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.adapter.RecordMessagingMessageListenerAdapter.onMessage(RecordMessagingMessageListenerAdapter.java:53) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeOnMessage(KafkaMessageListenerContainer.java:2653) ~[spring-kafka-2.8.7.jar:2.8.7]
    ... 12 common frames omitted
```

브로커는 commit 신호를 받아서 해당 토픽, 파티션의 offset을 증가시킨다.  
컨수머 입장에서는 해당 데이터 처리에 실패했지만, 브로커 입장에서는 정상적으로 데이터가 처리됐을 때와 마찬가지로 commit 신호를 받으므로 장애는 발생하지 않는다.


## 역직렬화에 실패하면..

그런데 2번 역직렬화 과정에서 에러가 나면 어떻게 될까?

예를 들어 JsonSerializer/JsonDeserializer를 사용해서 JSON으로 직렬화/역직렬화하도록 구성했는데,  
JSON 규격에 맞지 않는 메시지(poison pill)가 인입되면 역직렬화 과정에서 에러가 발생하게 된다.

2번 **역직렬화 과정은 3번 과정보다 앞서 진행되므로 2번 과정에서 에러가 발생하면 개발자가 작성한 리스너로 진입하지 못하며,**  
그 앞 단계에서 즉 프레임워크 수준에서 에러 처리가 돼야 한다.  
그런데 이 에러 처리를 별로 신경쓰지 않고 사용하다가 역직렬화 과정에서 에러가 발생하면,  
3번 과정에서 처럼 정해진 횟수만큼 재시도하다가 얌전히 포기하고 commit 처리되는 게 아니라,  
**역직렬화에 성공할 때까지 계속, 그것도 아주 빠른 속도로 재시도한다.**

그런데 역직렬화 실패한 데이터를 다시 역직렬화 하면 성공할 수 있을까?  
그럴 가능성은 거의 없다고 생각하는데 안타깝게도 카프카 컨수머는, 아니 최소한 **카프카 스프링 컨수머는 역직렬화 안 되면 될 때까지, 서버 빠개질때까지!! 그렇게 무식하게 동작한다.**

그 결과 컨수머 시스템 자원이 갉아먹히는데,  
**이 컨수머가 다른 토픽의 컨수머이기도 하면 자원 부족으로 그 다른 토픽의 메시지를 컨숨하는 속도가 느려지고 그 다른 토픽에도 메시지가 쌓이게 된다.**  
**결국 이 컨수머를 포함하고 있는 서버 인스턴스들은 점점 좀비화되고 서비스 장애 상황으로 이어진다.**

그러니 역직렬화에 실패하면,  
마치 혁명에 실패하면 반역죄로 능지처사에 3족이 멸문 당하는 것처럼 아주 걍 작살이 나는 거시다.


## Spring ErrorHandlingDeserializer

위에 '(역직렬화 과정) 에러 처리를 별로 신경쓰지 않고 기본 설정으로 두고 사용하면'이라는 단서가 있는데,  
바꿔말하면 역직렬화 에러 처리를 신경쓰면 위와 같은 장애 발생을 막을 수 있다. 어떻게?

**스프링 카프카 2.2에 도입된 ErrorHandlingDeserializer를 사용하고, ErrorHandler를 지정해주면 된다.**

이 내용은 역직렬화 재시도를 계속하면서 펑펑 싸대는 아래와 같은 로그에 나름 친절하게 나와있다.

```
2022-08-27 14:03:22.077 ERROR 81052 --- [ntainer#0-0-C-1] o.s.k.l.KafkaMessageListenerContainer    : Consumer exception

java.lang.IllegalStateException: This error handler cannot process 'SerializationException's directly; please consider configuring an 'ErrorHandlingDeserializer' in the value and/or key deserializer
    at org.springframework.kafka.listener.DefaultErrorHandler.handleOtherException(DefaultErrorHandler.java:149) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.handleConsumerException(KafkaMessageListenerContainer.java:1799) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.run(KafkaMessageListenerContainer.java:1298) ~[spring-kafka-2.8.7.jar:2.8.7]
    at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:264) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run(FutureTask.java) ~[na:na]
    at java.base/java.lang.Thread.run(Thread.java:832) ~[na:na]
Caused by: org.apache.kafka.common.errors.RecordDeserializationException: Error deserializing key/value for partition basic-topic-01-2 at offset 5. If needed, please seek past the record to continue consumption.
    at org.apache.kafka.clients.consumer.internals.Fetcher.parseRecord(Fetcher.java:1448) ~[kafka-clients-3.1.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.access$3400(Fetcher.java:135) ~[kafka-clients-3.1.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.fetchRecords(Fetcher.java:1671) ~[kafka-clients-3.1.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.access$1900(Fetcher.java:1507) ~[kafka-clients-3.1.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.fetchRecords(Fetcher.java:733) ~[kafka-clients-3.1.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.fetchedRecords(Fetcher.java:684) ~[kafka-clients-3.1.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.pollForFetches(KafkaConsumer.java:1277) ~[kafka-clients-3.1.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1238) ~[kafka-clients-3.1.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1211) ~[kafka-clients-3.1.1.jar:na]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollConsumer(KafkaMessageListenerContainer.java:1522) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doPoll(KafkaMessageListenerContainer.java:1512) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollAndInvoke(KafkaMessageListenerContainer.java:1340) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.run(KafkaMessageListenerContainer.java:1252) ~[spring-kafka-2.8.7.jar:2.8.7]
    ... 4 common frames omitted
Caused by: java.lang.IllegalStateException: No type information in headers and no default type provided
    at org.springframework.util.Assert.state(Assert.java:76) ~[spring-core-5.3.21.jar:5.3.21]
    at org.springframework.kafka.support.serializer.JsonDeserializer.deserialize(JsonDeserializer.java:583) ~[spring-kafka-2.8.7.jar:2.8.7]
    at org.apache.kafka.clients.consumer.internals.Fetcher.parseRecord(Fetcher.java:1439) ~[kafka-clients-3.1.1.jar:na]
    ... 16 common frames omitted

```

물론 공식 문서에도 나와있다.  
(하지만 우리는 에러가 발생하기 전까지는 공식 문서를 잘 안..=3=3)

어쨌든 ErrorHandlingDeserializer 설정 방법이나 알아보자.


### ErrorHandlingDeserializer

스프링 카프카 설정 방법은 여러가지인데 결국에는 KafkaConsumerFactory에 ErrorHandlingDeserializer를 지정해주면 된다.

소스 코드로는 대략 다음과 같이 작성하면 되고,

```kotlin
// from https://docs.spring.io/spring-kafka/reference/html/#error-handling-deserializer

... // other props
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);

props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, JsonDeserializer.class);
props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class.getName());

return DefaultKafkaConsumerFactory<>(props);
```

yml로 다음과 같이 작성해도 효과는 같다.

```yml
spring:  
  kafka:
    consumer:
      bootstrap-servers:
        - localhost:29092
      key-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring:
          deserializer:
            key:
              delegate:
                class: org.springframework.kafka.support.serializer.JsonDeserializer
            value:
              delegate:
                class: org.springframework.kafka.support.serializer.JsonDeserializer
```

결국 **ErrorHandlingDeserializer로 JsonDeserializer를 감싸서, 에러가 있으면 ErrorHandler에게 넘기고, 에러가 없으면 JsonDeserializer에게 넘기는 구조다.**


### ErrorHandler

에러 핸들러 지정 방식도 여러가지인데 가장 간단하게는 KafkaListenerContainerFactory에 ErrorHandler를 지정해주면 된다.

```kotlin
    @Bean
    fun kafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, String> {
        
        val factory = ConcurrentKafkaListenerContainerFactory<String, String>()
        
        // ErrorHandler 지정
        factory.setCommonErrorHandler(                  
            DefaultErrorHandler { record, exception ->
                log.error(
                    """
                        Consume 실패!!!
                        cause: ${exception.message}
                        topic: ${record.topic()}
                        partition: ${record.partition()}
                        offset: ${record.offset()}
                        message key: ${record.key()}
                        message value: ${record.value()}
                        ${exception.stackTraceToString()}
                    """.trimIndent()
                )
            }
        )

        factory.consumerFactory = consumerFactory()  // 앞서 ErrorHandlingDeserializer 설정할 때 만든 DefaultKafkaConsumerFactory

        return factory
    }
```

단순히 로깅만으로 끝내도 되면 위와 같이 간단히 처리할 수 있고, 그게 아니라면 dead-letter topic으로 전송되게 할 수도 있다.

dead-letter topic 관련은 https://docs.spring.io/spring-kafka/reference/html/#dead-letters 를 참고하자.

이밖에도 @KafkaListener에서 에러 처리를 지정할 수도 있고,  
`consumerProps.put(ErrorHandlingDeserializer.VALUE_FUNCTION, FailedFooProvider.class)` 이런 식으로 역직렬화에 실패한 메시지 대신에 다른 fallback 메시지를 만들어서 리스너에게 전달해줄 수도 있다.


## 마무리

>- **카프카 스프링 컨수머에서 역직렬화에 실패하면 완전히 새된다.**
>- StringDeserializer를 사용하면 역직렬화 실패 가능성은 매우 낮아지지만, 실제 비즈니스에서 사용하려면 역직렬화한 문자열을 다시 업무에 사용하는 데이터 타입으로 변환해줘야 한다.
>- 이 과정을 JSON으로 한 번에 해주는 게 JsonDeserializer
>- 하지만 JsonDeserializer를 날로 그냥 먹으면 심각한 식중독에 걸릴 수 있다.
>- **JsonDeserializer 사용 등 역직렬화 오류가 발생할 수 있는 상황에서는 반드시 ErrorHandlingDeserializer를 사용해야 Poison Pill로 인한 장애를 막을 수 있다.**

## 참고

- https://docs.spring.io/spring-kafka/reference/html/#error-handling-deserializer
- https://www.confluent.io/ko-kr/blog/spring-kafka-can-your-kafka-consumers-handle-a-poison-pill/


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
