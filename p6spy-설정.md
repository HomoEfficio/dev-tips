# p6spy 설정

## update 2021-05-08

- 스프링 부트용 서드파티 - https://github.com/gavlyukovskiy/spring-boot-data-source-decorator


### build.gradle.kts

`p6spy-spring-boot-starter`를 implementation 으로 지정해야함, developmentOnly 로 지정하면 P6SpyDriver 못 찾아서 오류

```kotlin
dependencies {
    implementation("com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.7.1")
}
```

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:p6spy:h2:mem:creator-in-app-game;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=TRUE
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver

decorator.datasource.p6spy:
  enable-logging: true
  multiline: true
  logging: slf4j
  tracing.include-parameter-values: true
```

### spy.properties

```
driverlist=org.h2.Driver
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=me.zepeto.creator.in_app_game.creator.command.config.P6SpyZepetoLineFormat
```
### Formatter

com.p6spy.engine.spy.appender.MultiLineFormat 참고해서 아래와 같은 구현체 만들어서 지정 가능

```kotlin
class P6SpyZepetoLineFormat: MessageFormattingStrategy {
    override fun formatMessage(
        connectionId: Int,
        now: String?,
        elapsed: Long,
        category: String?,
        prepared: String?,
        sql: String?,
        url: String?
    ): String {
        return """#$now | took ${elapsed}ms | $category | connection $connectionId| url $url
$sql;"""
    }
}
```


---

JDBC 를 통해 DB로 전달되는 쿼리를 볼 수 있게 해주는 라이브러리인 p6spy를 Spring Boot + JPA 환경에서 사용하는 방법을 알아보자.

## Hibernate 쿼리도 있는데 뭐하러?

[여기](https://github.com/HomoEfficio/dev-tips/blob/master/JPA%20로깅%20설정.md)에 정리해놓은 것처럼 Hibernate에서도 적절히 설정을 하면 쿼리를 볼 수는 있는데 다음과 같이 쿼리와 파라미터가 통합되지 않고 분리되어 표시되므로 가독성이 그리 좋지 않다.

```
Hibernate: 
    insert 
    into
        product
        (category_id, name, price, product_id) 
    values
        (?, ?, ?, ?)
2018-07-29 09:39:35.933 TRACE 15228 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [BIGINT] - [1]
2018-07-29 09:39:35.934 TRACE 15228 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [VARCHAR] - [라텍스 밴드 중급형]
2018-07-29 09:39:35.937 TRACE 15228 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [3] as [DOUBLE] - [28.0]
2018-07-29 09:39:35.938 TRACE 15228 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [4] as [BIGINT] - [2]
```

위 `?` 자리에 파라미터를 그냥 표시해주면 훨씬 나을 것 같다. p6spy는 바로 이 가려운 부분을 시원하게 긁어준다.

## 설정

### 의존관계 추가

mvnrepository.com 에서 p6spy로 검색해서 build.gradle 파일에 다음과 같이 추가해서 의존관계를 추가한다.

```groovy
...
dependencies {
    ...
    implementation group: 'p6spy', name: 'p6spy', version: '3.8.6'
    ...
}
...
```

### DataSource 설정

application.yml 파일에서 `url`과 `driver-class-name` 항목을 다음과 같이 p6spy 기준으로 변경한다.

- url: `jdbc:h2:...` -> `jdbc:p6spy:h2:...`
- driver-class-name: com.p6spy.engine.spy.P6SpyDriver 로 변경 (실제 DB 드라이버는 다른 곳에 설정)

```yml
spring:
  datasource:
    url: jdbc:p6spy:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
```

### spy.properties 파일 작성

resources 폴더 아래에 다음과 같이 spy.properties 파일을 작성한다.

```properties
driverlist=org.h2.Driver  # <-- 실제 드라이버!!
appender=com.p6spy.engine.spy.appender.StdoutLogger
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
```

spy.properties 에서 상당히 많은 항목을 설정할 수 있다. 자세한 내용은 [공식 문서](https://p6spy.readthedocs.io/en/latest/configandusage.html#common-property-file-settings)를 참고하고, 테스트 환경에서 콘솔 로그로 확인하려면 위와 같이 설정하면 충분하다.

## `@AutoconfigureTestDatabase` 설정

검색해보면 위와 같이 3가지 설정에 대한 내용만 나온다. 그래서 저렇게만 하면 콘솔 로그에 바로 찍힐 것 같지만 테스트 환경에서는, 특히 `@DataJpaTest`로 테스트를 실행하는 환경에서는 위 3가지만으로는 부족하다.

왜냐하면 `@DataJpaTest`는 다음과 같은 여러 애노테이션을 포함하는데,

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@BootstrapWith(DataJpaTestContextBootstrapper.class)
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)
@TypeExcludeFilters(DataJpaTypeExcludeFilter.class)
@Transactional
@AutoConfigureCache
@AutoConfigureDataJpa
@AutoConfigureTestDatabase  // <-- 여기!!
@AutoConfigureTestEntityManager
@ImportAutoConfiguration
public @interface DataJpaTest {
    ...
```

이 중에서 `@AutoConfigureTestDatabase` 이 애노테이션은 다음과 같이 기존에 설정되어 있는 데이터베이스 설정 대신에 테스트용 인메모리를 강제로 사용하는 것(Replace.ANY)이 기본 설정으로 되어 있다.

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@ImportAutoConfiguration
@PropertyMapping("spring.test.database")
public @interface AutoConfigureTestDatabase {

    /**
     * Determines what type of existing DataSource beans can be replaced.
     * @return the type of existing DataSource to replace
     */
    @PropertyMapping(skip = SkipPropertyMapping.ON_DEFAULT_VALUE)
    Replace replace() default Replace.ANY;  // <-- 여기

...
```

따라서, p6spy 드라이버를 사용하도록 설정된 내용이 무시되므로 p6spy가 만들어주는 쿼리가 콘솔에 표시되지 않는다.

그래서 **`@AutoconfigureTestDatabase`가 포함되는 애노테이션이 사용되는 테스트 환경에서는 다음과 같이 기본 설정값을 `Replace.NONE`로 변경해야 p6spy가 의도했던 대로 제대로 동작한다.**

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // <- 여기!!
@ActiveProfiles("test")
class MemberRepositoryTest {
```

```
...
#1573291447793 | took 1ms | statement | connection 0| url jdbc:p6spy:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
insert into member (id, email, password, username) values (null, ?, ?, ?)
insert into member (id, email, password, username) values (null, 'homo.efficio@gmail.com', 'Password1!', 'Homo Efficio');
...
```
