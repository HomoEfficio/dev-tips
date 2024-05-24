# Spring Conditional

로컬에서 애플리케이션을 실행했는데 카프카 컨수머로 등록되면서 와라락 메시지를 conume 하는 걸보고 화들짝 놀란 적이 있다면~~

프로파일에 `local`이라는 글자가 포함돼 있을 때는 컨수머 빈을 등록하지 않게 하면 좋겠다.

어떻게?

`Condition` 인터페이스의 구현체를 적당히 만들어서 `@Conditional`로 지정해주면 된다.


```kotlin
class NoLocalProfileCondition : Condition {
    override fun matches(context: ConditionContext, metadata: AnnotatedTypeMetadata): Boolean {
        return (context.environment.getProperty("spring.profiles.active") ?: "")
            .split(',')
            .all { !it.contains("local") }
    }
}


@Conditional(NoLocalProfileCondition::class)
@Configuration
class KafkaConsumerConfig(...) {
  // 이하 생략    
}


@Conditional(NoLocalProfileCondition::class)
@Component
class AbcEventListener(...) {
    @KafkaListener(
        topics = ["${KafkaTopic.ABC_EVENT}#{affixProvider.kafkaSuffix()}"],
        groupId = "${KafkaGroup.XYZ_ABC}#{affixProvider.kafkaSuffix()}",
        containerFactory = "kafkaListenerContainerFactory"
    )
    fun consume(record: ConsumerRecord<String, String>, ack: Acknowledgment) {

    // 이하 생략
}
```

참고로 처음에는 별도의 NoLocalProfileCondition 클래스를 만들지 않고 그냥 아래와 같이 `@ConditionalOnExpression + SpEL`로 해보려고 했으나,

```kotlin
@ConditionalOnExpression("#{T(java.util.Arrays).asList('\${spring.profiles.active}'.split(',')).stream().allMatch(p -> !p.contains('local')))
```

SpEL은 람다식을 지원하지 않아 `->`와 같은 문법을 지원하지 않는다는 사실을 한참 삽질 후에 알게 됐다.

빈 생성 등을 조건적으로 하고 싶은데 SpEL이든 뭐든 잘 안 되면 헤매지 말고 그냥 `@Conditional(Condition 인터페이스 구현체)`를 떠올리자

