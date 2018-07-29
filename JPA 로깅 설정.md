# JPA 로깅 설정

## 기본 설정

아래와 같이 최소한의 설정으로 시작해보자.

```yml
spring:
  datasource:
    url: jdbc:h2:mem:crypto-mall;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
  jpa:
    properties.hibernate:
      dialect: org.hibernate.dialect.MySQL5Dialect
```

이 상태에서 애플리케이션에서 insert 를 실행하면 다음과 같이 SQL 문이 타임스탬프 없이 표시되고 , 바인딩 값은 ? 로 표시된다.

```
...
2018-07-29 09:32:03.316  INFO 6184 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2018-07-29 09:32:04.039  INFO 6184 --- [           main] i.h.e.c.e.p.r.ProductRepositoryTest      : Started ProductRepositoryTest in 4.499 seconds (JVM running for 6.549)
2018-07-29 09:32:04.087  INFO 6184 --- [           main] o.s.t.c.transaction.TransactionContext   : Began transaction (1) for test context [DefaultTestContext@1b66c0fb testClass = ProductRepositoryTest, testInstance = io.homo.efficio.cryptomall.entity.product.repository.ProductRepositoryTest@2f67a4d3, testMethod = whenFindByName__thenReturnProduct@ProductRepositoryTest, testException = [null], mergedContextConfiguration = [MergedContextConfiguration@3e0e1046 testClass = ProductRepositoryTest, locations = '{}', classes = '{class io.homo.efficio.cryptomall.entity.CryptoMallEntityApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.autoconfigure.OverrideAutoConfigurationContextCustomizerFactory$DisableAutoConfigurationContextCustomizer@2892dae4, org.springframework.boot.test.autoconfigure.filter.TypeExcludeFiltersContextCustomizer@351584c0, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@1150bf5c, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@11bd0f3b, [ImportsContextCustomizer@24c1b2d2 key = [org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration, org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration, org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration, org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration, org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration, org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration, org.springframework.boot.test.autoconfigure.jdbc.TestDatabaseAutoConfiguration, org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManagerAutoConfiguration]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@2fd1433e, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@212b5695, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map[[empty]]]; transaction manager [org.springframework.orm.jpa.JpaTransactionManager@7e0f9528]; rollback [true]
Hibernate: select next_val as id_val from hibernate_sequence for update
Hibernate: update hibernate_sequence set next_val= ? where next_val=?
Hibernate: select next_val as id_val from hibernate_sequence for update
Hibernate: update hibernate_sequence set next_val= ? where next_val=?
Hibernate: insert into category (name, category_id) values (?, ?)
Hibernate: insert into product (category_id, name, price, product_id) values (?, ?, ?, ?)
...
```

## 바인딩 값 표시

아래와 같이 `logging.level.org.hibernate.type.descriptor.sql: TRACE`를 추가하면 바인딩 된 값을 볼 수 있다.

```yml
spring:
  datasource:
    url: jdbc:h2:mem:crypto-mall;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
  jpa:
    properties.hibernate:
      dialect: org.hibernate.dialect.MySQL5Dialect

logging.level.org.hibernate:
  type.descriptor.sql: TRACE  # 여기 추가!
```

```
2018-07-29 09:36:25.377  INFO 23648 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2018-07-29 09:36:26.134  INFO 23648 --- [           main] i.h.e.c.e.p.r.ProductRepositoryTest      : Started ProductRepositoryTest in 4.461 seconds (JVM running for 6.241)
2018-07-29 09:36:26.185  INFO 23648 --- [           main] o.s.t.c.transaction.TransactionContext   : Began transaction (1) for test context [DefaultTestContext@1b66c0fb testClass = ProductRepositoryTest, testInstance = io.homo.efficio.cryptomall.entity.product.repository.ProductRepositoryTest@2f67a4d3, testMethod = whenFindByName__thenReturnProduct@ProductRepositoryTest, testException = [null], mergedContextConfiguration = [MergedContextConfiguration@3e0e1046 testClass = ProductRepositoryTest, locations = '{}', classes = '{class io.homo.efficio.cryptomall.entity.CryptoMallEntityApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.autoconfigure.OverrideAutoConfigurationContextCustomizerFactory$DisableAutoConfigurationContextCustomizer@2892dae4, org.springframework.boot.test.autoconfigure.filter.TypeExcludeFiltersContextCustomizer@351584c0, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@1150bf5c, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@11bd0f3b, [ImportsContextCustomizer@24c1b2d2 key = [org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration, org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration, org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration, org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration, org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration, org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration, org.springframework.boot.test.autoconfigure.jdbc.TestDatabaseAutoConfiguration, org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManagerAutoConfiguration]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@2fd1433e, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@212b5695, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map[[empty]]]; transaction manager [org.springframework.orm.jpa.JpaTransactionManager@5f56424d]; rollback [true]
Hibernate: select next_val as id_val from hibernate_sequence for update
Hibernate: update hibernate_sequence set next_val= ? where next_val=?
Hibernate: select next_val as id_val from hibernate_sequence for update
Hibernate: update hibernate_sequence set next_val= ? where next_val=?
Hibernate: insert into category (name, category_id) values (?, ?)
2018-07-29 09:36:26.348 TRACE 23648 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [VARCHAR] - [헬스용품]
2018-07-29 09:36:26.350 TRACE 23648 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [BIGINT] - [1]
Hibernate: insert into product (category_id, name, price, product_id) values (?, ?, ?, ?)
2018-07-29 09:36:26.354 TRACE 23648 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [BIGINT] - [1]
2018-07-29 09:36:26.354 TRACE 23648 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [VARCHAR] - [라텍스 밴드 중급형]
2018-07-29 09:36:26.355 TRACE 23648 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [3] as [DOUBLE] - [28.0]
2018-07-29 09:36:26.356 TRACE 23648 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [4] as [BIGINT] - [2]
```


## SQL 문 포맷팅

아래와 같이 `spring.jpa.properties.hibernate.format_sql: true`를 추가하면,

```yml
spring:
  datasource:
    url: jdbc:h2:mem:crypto-mall;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
  jpa:
    properties.hibernate:
      dialect: org.hibernate.dialect.MySQL5Dialect
      format_sql: true  # 여기 추가!

logging.level.org.hibernate:
  type.descriptor.sql: TRACE
```

다음과 같이 보기 좋게 포맷팅 된 SQL 문을 볼 수 있다.

```
...
2018-07-29 09:39:34.956  INFO 15228 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2018-07-29 09:39:35.693  INFO 15228 --- [           main] i.h.e.c.e.p.r.ProductRepositoryTest      : Started ProductRepositoryTest in 4.308 seconds (JVM running for 6.255)
2018-07-29 09:39:35.743  INFO 15228 --- [           main] o.s.t.c.transaction.TransactionContext   : Began transaction (1) for test context [DefaultTestContext@1b66c0fb testClass = ProductRepositoryTest, testInstance = io.homo.efficio.cryptomall.entity.product.repository.ProductRepositoryTest@2f67a4d3, testMethod = whenFindByName__thenReturnProduct@ProductRepositoryTest, testException = [null], mergedContextConfiguration = [MergedContextConfiguration@3e0e1046 testClass = ProductRepositoryTest, locations = '{}', classes = '{class io.homo.efficio.cryptomall.entity.CryptoMallEntityApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.autoconfigure.OverrideAutoConfigurationContextCustomizerFactory$DisableAutoConfigurationContextCustomizer@2892dae4, org.springframework.boot.test.autoconfigure.filter.TypeExcludeFiltersContextCustomizer@351584c0, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@1150bf5c, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@11bd0f3b, [ImportsContextCustomizer@24c1b2d2 key = [org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration, org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration, org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration, org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration, org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration, org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration, org.springframework.boot.test.autoconfigure.jdbc.TestDatabaseAutoConfiguration, org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManagerAutoConfiguration]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@2fd1433e, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@212b5695, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map[[empty]]]; transaction manager [org.springframework.orm.jpa.JpaTransactionManager@448cdb47]; rollback [true]
Hibernate: 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
Hibernate: 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
Hibernate: 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
Hibernate: 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
Hibernate: 
    insert 
    into
        category
        (name, category_id) 
    values
        (?, ?)
2018-07-29 09:39:35.920 TRACE 15228 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [VARCHAR] - [헬스용품]
2018-07-29 09:39:35.923 TRACE 15228 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [BIGINT] - [1]
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
...
```

## 기타

### SQL 문에도 타임스탬프 표시

아래와 같이 `logging.level.org.hibernate.SQL: DEGUB`를 추가하면 된다.

```yml
spring:
  datasource:
    url: jdbc:h2:mem:crypto-mall;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
  jpa:
    properties.hibernate:
      dialect: org.hibernate.dialect.MySQL5Dialect
      format_sql: true

logging.level.org.hibernate:
  SQL: DEBUG  # 여기 추가!
  type.descriptor.sql: TRACE
```

하지만 기존 내용에 DEBUG 로그를 추가하는 방식이라서 다음과 같이 SQL 문이 두 번씩 표시되는 단점이 있다.

```
...
2018-07-29 10:55:55.103  INFO 26536 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2018-07-29 10:55:55.865  INFO 26536 --- [           main] i.h.e.c.e.p.r.ProductRepositoryTest      : Started ProductRepositoryTest in 4.827 seconds (JVM running for 6.83)
2018-07-29 10:55:55.933  INFO 26536 --- [           main] o.s.t.c.transaction.TransactionContext   : Began transaction (1) for test context [DefaultTestContext@1b66c0fb testClass = ProductRepositoryTest, testInstance = io.homo.efficio.cryptomall.entity.product.repository.ProductRepositoryTest@2f67a4d3, testMethod = whenFindByName__thenReturnProduct@ProductRepositoryTest, testException = [null], mergedContextConfiguration = [MergedContextConfiguration@3e0e1046 testClass = ProductRepositoryTest, locations = '{}', classes = '{class io.homo.efficio.cryptomall.entity.CryptoMallEntityApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.autoconfigure.OverrideAutoConfigurationContextCustomizerFactory$DisableAutoConfigurationContextCustomizer@2892dae4, org.springframework.boot.test.autoconfigure.filter.TypeExcludeFiltersContextCustomizer@351584c0, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@1150bf5c, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@11bd0f3b, [ImportsContextCustomizer@24c1b2d2 key = [org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration, org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration, org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration, org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration, org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration, org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration, org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration, org.springframework.boot.test.autoconfigure.jdbc.TestDatabaseAutoConfiguration, org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManagerAutoConfiguration]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@2fd1433e, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@212b5695, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map[[empty]]]; transaction manager [org.springframework.orm.jpa.JpaTransactionManager@550c973e]; rollback [true]
2018-07-29 10:55:56.042 DEBUG 26536 --- [           main] org.hibernate.SQL                        : 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
Hibernate: 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
2018-07-29 10:55:56.043 DEBUG 26536 --- [           main] org.hibernate.SQL                        : 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
Hibernate: 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
2018-07-29 10:55:56.070 DEBUG 26536 --- [           main] org.hibernate.SQL                        : 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
Hibernate: 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
2018-07-29 10:55:56.071 DEBUG 26536 --- [           main] org.hibernate.SQL                        : 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
Hibernate: 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
2018-07-29 10:55:56.087 DEBUG 26536 --- [           main] org.hibernate.SQL                        : 
    insert 
    into
        category
        (name, category_id) 
    values
        (?, ?)
Hibernate: 
    insert 
    into
        category
        (name, category_id) 
    values
        (?, ?)
2018-07-29 10:55:56.092 TRACE 26536 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [VARCHAR] - [헬스용품]
2018-07-29 10:55:56.093 TRACE 26536 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [BIGINT] - [1]
2018-07-29 10:55:56.096 DEBUG 26536 --- [           main] org.hibernate.SQL                        : 
    insert 
    into
        product
        (category_id, name, price, product_id) 
    values
        (?, ?, ?, ?)
Hibernate: 
    insert 
    into
        product
        (category_id, name, price, product_id) 
    values
        (?, ?, ?, ?)
2018-07-29 10:55:56.096 TRACE 26536 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [BIGINT] - [1]
2018-07-29 10:55:56.096 TRACE 26536 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [VARCHAR] - [라텍스 밴드 중급형]
2018-07-29 10:55:56.097 TRACE 26536 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [3] as [DOUBLE] - [28.0]
2018-07-29 10:55:56.098 TRACE 26536 --- [           main] o.h.type.descriptor.sql.BasicBinder      : binding parameter [4] as [BIGINT] - [2]
...
```

### JPA 구현체의 내부 동작 표시

다음과 같이 `type.descriptor.sql: TRACE` 대신 `type: TRACE`로 설정하면 아래와 같이 JPA 구현체가 내부적으로 수행하는 동작에 대한 로그도 볼 수 있다.

```yml
spring:
  datasource:
    url: jdbc:h2:mem:crypto-mall;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
  jpa:
    properties.hibernate:
      dialect: org.hibernate.dialect.MySQL5Dialect
      format_sql: true

logging.level.org.hibernate:
  SQL: DEBUG
#  type.descriptor.sql: TRACE
  type: TRACE  # 이렇게 변경
```

```
...
... Adding type registration url -> org.hibernate.type.UrlType@411a5965
... Adding type registration java.net.URL -> org.hibernate.type.UrlType@411a5965
... Adding type registration Duration -> org.hibernate.type.DurationType@5d5b9ecb
... Adding type registration java.time.Duration -> org.hibernate.type.DurationType@5d5b9ecb
... Adding type registration Instant -> org.hibernate.type.InstantType@47311277
... Adding type registration java.time.Instant -> org.hibernate.type.InstantType@47311277
... Adding type registration LocalDateTime -> org.hibernate.type.LocalDateTimeType@41b13f3d
... Adding type registration java.time.LocalDateTime -> org.hibernate.type.LocalDateTimeType@41b13f3d
...
```

JPA 구현체의 내부 동작을 구경하고 싶으면 위와 같이 해도 좋지만, 너무 많은 로그가 표시되므로 가끔 필요할 때 확인하는 용도로만 쓰는 것이 좋다.
