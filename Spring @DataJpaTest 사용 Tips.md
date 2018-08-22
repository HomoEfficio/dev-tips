# Spring `@DataJpaTest` 사용 Tips

스프링에서 `@DataJpaTest`로 Persistence를 테스트할 때 가끔 마주치는 에러 대처법

## Repository 클래스를 못 찾을 때

대략 다음과 같은 에러가 난다.

>Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'a.b.c.MyRepository' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {@org.springframework.beans.factory.annotation.Autowired(required=true)}

Repository를 못 찾아서 나는 에러인데 다음과 같이 `@EnableJpaRepositories` 애노테이션을 써서 명시적으로 Repository 위치를 명시적으로 지정해주면 된다.

```java
@RunWith(SpringRunner.class)
@DataJpaTest
@EnableJpaRepositories("a.b.c")  // <- 이거!!
public class MyRepositoryTest {
```

## 엔티티를 못 찾을 때

대략 다음과 같은 에러가 난다.

>Caused by: java.lang.IllegalArgumentException: Not a managed type: class a.b.c.My

`Not a managed type`이라고 되어 있어서 `@Entity`가 안 붙어 있나.. 하고 해당 클래스를 봤는데 `@Entity`가 붙어있다면 이건 에러 메시지와는 다르게 Managed가 되지 않고 있어서 발생한 에러가 아니라 아예 해당 엔티티 클래스를 찾지 못해서 발생한 에러다.

이럴 때는 다음과 같이 `@EntityScan` 애노테이션을 써서 명시적으로 엔티티 클래스 위치를 지정해주면 된다.

```java
@RunWith(SpringRunner.class)
@DataJpaTest
@EnableJpaRepositories("a.b.c")
@EntityScan("a.b.c")  // <- 이거!!
public class MyRepositoryTest {
```

## Embedded DB 말고 실제 DB를 사용하고 싶을 때

`@DataJpaTest`를 사용하면 내 의지와는 관계 없이 강제로 In-memory DB를 사용하게 된다. 아무리 실제 DB를 DataSource로 설정한 프로파일을 지정해줘도 안 통하고 다음과 같이 Embedded DB로 대체한다는 로그가 찍히고, 실제로도 h2 in-memory DB로 연결 된다.

>2018-08-22 12:45:46.322  INFO 85135 --- [           main] beddedDataSourceBeanFactoryPostProcessor : Replacing 'dataSource' DataSource bean with embedded version  <<==== 여기!!
>
>...
>
>2018-08-22 12:45:46.596  INFO 85135 --- [           main] o.s.j.d.e.EmbeddedDatabaseFactory        : Starting embedded database: url='jdbc:h2:mem:15e04945-1778-4116-a144-d133aadb445b;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=false', username='sa'  <<==== 여기!!
>
>...

[DataJpaTest](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/autoconfigure/orm/jpa/DataJpaTest.html) 문서에 보면 다음과 같이 설명되어 있다.

>By default, tests annotated with @DataJpaTest will use an embedded in-memory database (replacing any explicit or usually auto-configured DataSource). The @AutoConfigureTestDatabase annotation can be used to override these settings.

그래서 다음과 같이 `@AutoConfigureTestDatabase` 애노테이션으로 강제 대체를 막으면 application.properties 파일에 설정된 DB로 연결할 수 있다.

```java
@RunWith(SpringRunner.class)
@DataJpaTest
@EnableJpaRepositories("a.b.c")
@EntityScan("a.b.c")
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)  // <- 이거!!
public class MyRepositoryTest {
```
