# Spring Data MongoDB Logging

스프링 데이터 몽고디비를 사용할 때 실제 어떤 쿼리문이 전송되는지 알려면 로깅 수준을 DEBUG로 설정해서 로그를 확인하면 된다.

```yml
logging:
  level:
    org.springframework.data.mongodb.core.MongoTemplate: DEBUG  # MongoTemplate 사용 시
    org.springframework.data.mongodb.repository.query: DEBUG  # Repository 사용 시
```

예를 들어 도메인 클래스에 `@Field`를 사용해서 클래스 필드명과 컬렉션 필드명을 다르게 구성한다면 어떻게 될까?

```kotlin
@Document(collection = "daily_project_stat")
data class DailyProjectStat(

    @Field("project")
    val projectId: String,  // 여기!!

    val statName: StatType,

    val statDate: Instant,

    val statValue: Long,

    val createdAt: Instant? = null,

    val updatedAt: Instant? = null,
) {
}
```

다음과 같은 리포지토리를 정의해서

```kotlin
interface DailyProjectStatRepository : MongoRepository<DailyProjectStat, String> {

    fun findAllByProjectIdAndStatNameAndStatDateBetweenOrderByStatDateAsc(
        projectId: String,
        statName: StatType,
        from: Instant,
        to: Instant,
    ): List<DailyProjectStat>
}
```

위 메서드로 쿼리를 날려보면 다음과 같이 로그에 출력된다.

```
{ "projectId" : "xxx.yyy.zzz", "statName" : { "$java" : TOTAL_USER }, "statDate" : { "$gt" : { "$java" : 2023-10-08T14:59:59.999Z }, "$lt" : { "$java" : 2023-10-21T15:00:00.001Z } } }, Fields: {}, Sort: { "statDate" : 1}
```

처음에는 조회 결과가 의도한대로 나오지 않아 로그에 표시된 필드명(projectId)이 몽고디비 컬렉션의 필드명(project)과 맞지 않고, 객체 타입에 대해서는 `$java`라는 필드가 포함돼 있는 것이 원인인 줄 알았는데, 다른 이유가 있었다.  

[spring.data.mongodb.field-naming-strategy](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.data.spring.data.mongodb.field-naming-strategy)를 지정해주지 않아서였다.
다음과 같이 snakecase 를 지정해주니 조회가 된다.

```yml
spring.data.mongodb.field-naming-strategy: org.springframework.data.mapping.model.SnakeCaseFieldNamingStrategy
```

해결하고 보니 위와 마찬가지로 몽고디비 컬렉션 필드명이 아니라 클래스의 필드명과 `$java`가 로그에 찍혀도 조회 결과가 제대로 나왔다.  
그러니 로그에 찍힌 조회 조건이 몽고디비에 그대로 전달되는 것은 아니고 중간에 어디에선가 한 번 더 변환을 해주는 것으로 보인다. 그게 어딘지는 나중에 기회되면 찾아보기로 ㅋㅋ

객체 파라미터를 사용할 때는 https://docs.spring.io/spring-data/mongodb/docs/3.4.17/reference/html/#mapping-chapter 를 참고해서 필요하다면 Custom Converter를 만들어줘야 한다.
위에서 사용된 StatType은 enum 클래스인데 별도의 컨버터를 등록해주지 않아도 문자열값으로 자동으로 변환해주는 것으로 보이며,  
Instant는 위 스프링 데이터 몽고디비 매핑 문서에 따르면 자동 변환을 해주지 않는 것으로 보이지만 실제로는 별도의 컨버터를 등록해주지 않아도 의도한대로 잘 동작한다.

