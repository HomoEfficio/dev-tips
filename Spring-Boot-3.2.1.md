# Spring Boot to 3.2.1

- https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide


## Spring Security from 5.7.3 to 5.8

- https://docs.spring.io/spring-security/reference/5.8/getting-spring-security.html

### WebSecurityConfigurerAdapter 상속 제거

- https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter/
- SecurityConfig
  - SecurityFilterChain 빈 추가

### GlobalMethodSecurityConfiguration 상속 제거

- https://docs.spring.io/spring-security/reference/5.8/servlet/authorization/method-security.html
- MethodSecurityConfig
  - `@EnableGlobalMethodSecurity` -> `@EnableMethodSecurity`
  - RoleHierarchy를 포함하는 MethodSecurityExpressionHandler 빈 추가


## Deprecated 메서드 호출 수정

- toLowerCase() -> lowercase()
- toUpperCase() -> uppercase()
- 필드 초기화 init -> 생성자에 할당
- BraveTracer(tracer, BraveBaggageManager()) -> BraveTracer(tracer, null, BraveBaggageManager())

## Gradle 8.5

### deprecated 수정

- https://docs.gradle.org/current/userguide/upgrading_version_8.html#project_builddir
  - val generatedQuerydslDir = "$buildDir/generated/source/kapt/querydsl" -> val generatedQuerydslDir = layout.buildDirectory.dir("/generated/source/kapt/querydsl")
- sourceSets["main"].withConvention(KotlinSourceSet::class) { -> kotlin.sourceSets.main {


## Spring Boot 3.2.1 적용

각 단계 별로 수행 후 컴파일 에러 등이 있을 수 있으나 계속 진행하면 결국에는 성공

### javax -> jakarta

- https://blog.jetbrains.com/idea/2021/06/intellij-idea-eap-6/#automatic_migration_from_java_ee_to_jakarta_ee
  - Refactor > Migrate Packages and Classes > Java EE to Jakarta EE > Scope: Whole Project > Run
- implementation("javax.servlet:javax.servlet-api") -> implementation("jakarta.servlet:jakarta.servlet-api")
- allOpen > annotation("javax.persistence.Entity") -> annotation("jakarta.persistence.Entity") 나머지 2가지도 jakarta 로 변경
- 이 때까지는 아직 jakarta 의존관계가 추가돼 있지 않아 컴파일 에러인 상태 다음 단계에서 스프링 부트 3 의존 관계를 지정하면서 컴파일 에러 해소됨
- javax.* 가 클래스 패스에 실제로 존재할 때(스프링 부트 3.2.1 적용 전) 수행해야 IDE가 빠짐 없이 탐색해서 변경이 필요한 javax.* 만 jakarta.* 로 변경해준다
- 코드는 jakarta.* 로 변경됐지만 아직 jakarta 가 클래스패스에 없으므로 컴파일 에러 발생하지만 스프링 부트 3이 적용된 후에는 jakarta 관련 컴파일 에러는 사라진다


### plugin 버전 지정 중앙화

- https://github.com/HomoEfficio/dev-tips/blob/master/Gradle-PluginManagement-plugin-version.md
- 중앙에서 springBootVersion 3.2.1, springBootDependencyManagementVersion 1.1.4 로 변경하고 리빌드하면 3.2.1 이 적용된다
- 이후 여러 컴파일 에러 발생


### Spring Cloud Sleuth 를 Micrometer Tracing 으로 교체

- https://techblog.lycorp.co.jp/ko/how-to-migrate-to-spring-boot-3
- import org.springframework.cloud.sleuth.Tracer -> import io.micrometer.tracing.Tracer
- import org.springframework.cloud.sleuth.brave.bridge.BraveBaggageManager -> import io.micrometer.tracing.brave.bridge.BraveBaggageManager
- import org.springframework.cloud.sleuth.brave.bridge.BraveTracer -> import io.micrometer.tracing.brave.bridge.BraveTracer

### QueryDsl

- https://velog.io/@juhyeon1114/Spring-QueryDsl-gradle-설정-Spring-boot-3.0-이상

### MySQL 의존관계

- runtimeOnly("mysql:mysql-connector-java") -> runtimeOnly("com.mysql:mysql-connector-j")

## Spring Security 5.8.9 지정 제거

- `ext["spring-security.version"]="5.8.9"` 제거하면 Spring Security 6.2.1 이 적용된다.


## Spring Security 6.2.1

- `@EnableWebSecurity` 에 `@Configuration` 추가
- DSL 사용

  ```kotlin
      @Bean
      fun filterChain(http: HttpSecurity): SecurityFilterChain {
          http
              .csrf { configurer ->
                  configurer.disable()
              }
              .httpBasic { configurer ->
                  configurer.disable()
              }
              .formLogin { configurer ->
                  configurer.disable()
              }
              .cors(Customizer.withDefaults())
              .authorizeHttpRequests { authorizationManagerRequestMatcherRegistry ->
                  authorizationManagerRequestMatcherRegistry
                      .requestMatchers(HttpMethod.GET, "/").permitAll()
                      .requestMatchers(*swaggerAllowedList()).permitAll()
                      .requestMatchers(*actuatorAllowedList()).permitAll()
                      .requestMatchers(*probeAllowedList()).permitAll()
                      .requestMatchers(*devAllowedList()).permitAll()
                      .requestMatchers(*serverKeyAllowedList()).anonymous()
                      .anyRequest().authenticated()
              }
              .headers { headersConfigurer ->
                  headersConfigurer
                      .addHeaderWriter(XFrameOptionsHeaderWriter(XFrameOptionsHeaderWriter.XFrameOptionsMode.SAMEORIGIN))
              }
              .addFilterBefore(
                  ErrorResolvingFilter(handlerExceptionResolver),
                  ChannelProcessingFilter::class.java
              )
              .addFilterBefore(JwtAuthFilter(jwtProcessor), UsernamePasswordAuthenticationFilter::class.java)
              .addFilterAfter(
                  ServerKeyAuthFilter(
                      serverKeyAuthenticator = ServerKeyAuthenticatorImpl(serverKeyRepository),
                      serverType = ServerType.INTERNAL,
                      subsystemType = SubsystemType.ADMIN,
                      exemptionList = listOf("/external-api/**"),
                      jwtProcessor
                  ),
                  UsernamePasswordAuthenticationFilter::class.java
              )
              .addFilterAfter(
                  ServerKeyAuthFilter(
                      serverKeyAuthenticator = ServerKeyAuthenticatorImpl(serverKeyRepository),
                      serverType = ServerType.EXTERNAL,
                      subsystemType = SubsystemType.ADMIN,
                      exemptionList = listOf("/internal-api/**"),
                      jwtProcessor
                  ),
                  UsernamePasswordAuthenticationFilter::class.java
              )

          return http.build()
      }
  ```


## 컴파일 에러

### ConstructorBinding

- https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0.0-M2-Release-Notes#constructingbinding-no-longer-needed-at-the-type-level
- `@ConfigurationProperties`와 함께 사용하는 `@ConstructorBinding` 제거

### OkHttp3ClientHttpRequestFactory deprecated

- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/client/OkHttp3ClientHttpRequestFactory.html
  ```kotlin
    @Bean
    fun myRestTemplate(restTemplateBuilder: RestTemplateBuilder): RestTemplate {
        return restTemplateBuilder
            .requestFactory(JdkClientHttpRequestFactory::class.java)
            .setConnectTimeout(Duration.ofSeconds(connectionTimeout))
            .setReadTimeout(Duration.ofSeconds(readTimeout))
            .build()
    }
  ```

### Tracer

- import org.springframework.cloud.sleuth.Tracer -> import io.micrometer.tracing.Tracer
- import org.springframework.cloud.sleuth.brave.bridge.BraveBaggageManager -> import io.micrometer.tracing.brave.bridge.BraveBaggageManager
- import org.springframework.cloud.sleuth.brave.bridge.BraveTracer -> import io.micrometer.tracing.brave.bridge.BraveTracer

### HttpStatusCode

- ClientHttpResponse.statusCode 의 타입이 HttpStatus 가 아니라 HttpStatusCode

  ```kotlin
          return response.statusCode.series() == HttpStatus.Series.CLIENT_ERROR
              || response.statusCode.series() == HttpStatus.Series.SERVER_ERROR

  ===>
          return response.statusCode.is4xxClientError
                  || response.statusCode.is5xxServerError
  ```

### BulkOperations

- BulkOperations 에 정의된 메서드의 파라미터가 변경되어, 테스트 용으로 BulkOperations를 구현한 FakeBulkOperations 의 메서드 파라미터도 변경



### @ConstructorBinding

- @ConfigurationProperties 와 함꼐 사용되는 @ConstructorBinding 제거

  ```kotlin
  @ConfigurationProperties(prefix = "security.jwt")
  @ConstructorBinding

  ===>

  @ConfigurationProperties(prefix = "security.jwt")
  ```

### resilience4j

  ```kotlin
      implementation("io.github.resilience4j:resilience4j-spring-boot2")
      implementation("io.github.resilience4j:resilience4j-all")

  ===>

      implementation("io.github.resilience4j:resilience4j-spring-boot3:${resilience4jVersion}")
      implementation("io.github.resilience4j:resilience4j-all:${resilience4jVersion}")
  ```

### springdoc

- import org.springdoc.api.annotations.ParameterObject -> import org.springdoc.core.annotations.ParameterObject



## 런타임 에러

### enum

- Hibernate 6.2 부터 java enum은 데이터베이스의 enum 타입으로 저장
  - https://docs.jboss.org/hibernate/orm/6.2/migration-guide/migration-guide.html#ddl-implicit-datatype-enum
- 따라서 다음과 같은 에러 발생
  - Schema-validation: wrong column type encountered in column [xxx] in table [yyy]; found [varchar (Types#VARCHAR)], but expecting [enum ('aaa','bbb') (Types#ENUM)]

  ```kotlin
  @Enumerated(EnumType.STRING)

  ===>

  @Enumerated(EnumType.STRING)
  @Column(columnDefinition = "varchar(255)")
  ```

여기까지 하고 admin-command 실행 성공

### ehcache

- https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#ehcache3
- Caused by: java.lang.NoClassDefFoundError: javax/xml/bind/ValidationEventHandler 발생
- https://groups.google.com/g/ehcache-users/c/sKfxWuTpY-U
  ```kotlin
      implementation("org.ehcache:ehcache") {
          capabilities {
              requireCapability("org.ehcache:ehcache-jakarta")
          }
      }
  ```

### QueryDSL

- `fetchAll()` 사용 시 생성되는 `fetch all properties`에서 SyntaxException 발생
- 스프링 부트 3이 사용하는 하이버네이트 6에서는 `fetch all properties` 가 아예 없어져서 발생하는 오류
  - https://docs.jboss.org/hibernate/orm/6.0/migration-guide/migration-guide.html#_hql_fetch_all_properties_clause 
- `fetchAll()`을 제거하고 기능 정상 동작 여부 확인 필요



## 테스트 에러

### Security 관련 테스트

- 시큐리티 설정 시 기존 WebSecurityConfigurerAdapter를 상속받아 설정할 때는 WebMvcTest 시 SecurityConfig 를 명시적으로 `@Import`에 추가해주지 않아도 동작했지만,
- WebSecurityConfigurerAdapter를 상속받지 않고 SecurityFilterChain을 반환하는 빈 메서드를 사용하는 방식에서는 WebMvcTest 시 SecurityConfig 를 `@Import`에 명시적으로 추가해줘야 한다.

### OneToOne -> ManyToOne

- N:1 관계인데 실수로 OneToOne 으로 매핑돼 있어도 하이버네이트 5에서는 에러 발생하지 않으나, 하이버네이트 6에서는 다음과 같은 에러 발생

```
Caused by: org.springframework.dao.DataIntegrityViolationException: could not execute statement [Unique index or primary key violation: "PUBLIC.CONSTRAINT_INDEX_1 ON PUBLIC.XXX(YYY NULLS FIRST) VALUES ( /* 1 */ CAST(8 AS BIGINT) )"; SQL statement:
insert into xxx ...
```
- product:currency 는 N:1 관계인데 ManyToOne 이 아니라 OneToOne 으로 돼 있었음
- 하이버네이트 6에서는 OneToOne 인 경우 FK 컬럼에 unique 인덱스를 강제로 추가함
  - https://github.com/hibernate/hibernate-orm/blob/6.2/migration-guide.adoc#logical-1-1-unique
- 따라서 N:1 관계에 잘못 적용된 OneToOne을 ManyToOne 으로 수정하면 바로 해결

### okhttp 사용 제거

- Spring 6 에서 OkHttp 를 사용하지 않으므로 RestTemplate 을 대신 사용하도록 수정
- 기본 RestTemplate은 아직도 patch를 지원하지 않아 다음 에러 발생
  ```
  Caused by: java.net.ProtocolException: Invalid HTTP method: PATCH
    at java.base/java.net.HttpURLConnection.setRequestMethod(HttpURLConnection.java:489)
    at java.base/sun.net.www.protocol.http.HttpURLConnection.setRequestMethod(HttpURLConnection.java:598)
    at java.base/sun.net.www.protocol.https.HttpsURLConnectionImpl.setRequestMethod(HttpsURLConnectionImpl.java:344)
    at org.springframework.http.client.SimpleClientHttpRequestFactory.prepareConnection(SimpleClientHttpRequestFactory.java:201)
    at org.springframework.http.client.SimpleClientHttpRequestFactory.createRequest(SimpleClientHttpRequestFactory.java:156)
    at org.springframework.http.client.support.HttpAccessor.createRequest(HttpAccessor.java:124)
    at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:864)
  ```
- https://github.com/spring-projects/spring-boot/issues/33863 권고에 따라 httpclient5 추가

  ```kotlin
  implementation("org.apache.httpcomponents.client5:httpclient5")
  ```

### RestTemplate 에 ClientHttpRequestFactory 설정

- RestTemplate 을 다음과 같이 설정해서 사용하면, ZonedDateTime 데이터가 timestamp 형태로 변경되어 전송되기도 함
  ```kotlin
    @Bean
    fun myRestTemplate(): RestTemplate {
        return RestTemplateBuilder()
            .setConnectTimeout(Duration.ofSeconds(connectionTimeout))
            .setReadTimeout(Duration.ofSeconds(readTimeout))
            .build()
    }
  ```

- 다음과 같이 ClientHttpRequestFactory 설정을 추가해주면 원래의 datetime 형태로 전송됨

  ```kotlin
    @Bean
    fun myRestTemplate(restTemplateBuilder: RestTemplateBuilder): RestTemplate {
        return restTemplateBuilder
            .requestFactory(JdkClientHttpRequestFactory::class.java)
            .setConnectTimeout(Duration.ofSeconds(connectionTimeout))
            .setReadTimeout(Duration.ofSeconds(readTimeout))
            .build()
    }
  ```


### Kafka send 테스트 disable

- Embedded Kafka 설정 문제로 보이는데 해결할 수 없어 disable 처리함


### Flapdoodle Embedded MongoDB

- 스프링 3부터 flapdoodle embedded mongodb 지원 안함
  - https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#embedded-mongodb
- 별도 의존 관계 추가해줘야 함
  - https://github.com/flapdoodle-oss/de.flapdoodle.embed.mongo.spring


## spring-boot-properties-migrator

- 이거 하면 yml 파일 업그레이드 시 변경해야 할 부분을 알려준다고 하는데 넣고 실행해도 나오는 게 없었음




## 기타

### commons-logging

- 실행 시 맨 위에 아래와 깉은 로그가 찍힘

  ```
  Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts
  Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts
  21:24:00,796 |-INFO in ch.qos.logback.classic.LoggerContext[default] - This is logback-classic version 1.4.14
  ...
  ```
- 개별 서브 모듈의 build.gradle.kts 에 아래와 같이 commons-logging exclude 지정 추가
  - https://kdev.ing/spring-boot-commons-logging-conflicts/

  ```kotlin
  configurations.all {
      exclude(group = "commons-logging", module = "commons-logging")
  }
  ```

### logback 설정

- 실행 시 아래와 같은 로그가 찍힘

  ```
  Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts
  Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts
  21:24:00,796 |-INFO in ch.qos.logback.classic.LoggerContext[default] - This is logback-classic version 1.4.14
  21:24:00,798 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - Here is a list of configurators discovered as a service, by rank: 
  21:24:00,798 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 -   org.springframework.boot.logging.logback.RootLogLevelConfigurator
  21:24:00,798 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - They will be invoked in order until ExecutionStatus.DO_NOT_INVOKE_NEXT_IF_ANY is returned.
  21:24:00,798 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - Constructed configurator of type class org.springframework.boot.logging.logback.RootLogLevelConfigurator
  21:24:00,801 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - org.springframework.boot.logging.logback.RootLogLevelConfigurator.configure() call lasted 0 milliseconds. ExecutionStatus=INVOKE_NEXT_IF_ANY
  21:24:00,801 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - Trying to configure with ch.qos.logback.classic.joran.SerializedModelConfigurator
  21:24:00,801 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - Constructed configurator of type class ch.qos.logback.classic.joran.SerializedModelConfigurator
  21:24:00,804 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback-test.scmo]
  21:24:00,805 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.scmo]
  21:24:00,805 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - ch.qos.logback.classic.joran.SerializedModelConfigurator.configure() call lasted 4 milliseconds. ExecutionStatus=INVOKE_NEXT_IF_ANY
  21:24:00,805 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - Trying to configure with ch.qos.logback.classic.util.DefaultJoranConfigurator
  21:24:00,805 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - Constructed configurator of type class ch.qos.logback.classic.util.DefaultJoranConfigurator
  21:24:00,805 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback-test.xml]
  21:24:00,805 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.xml]
  21:24:00,805 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - ch.qos.logback.classic.util.DefaultJoranConfigurator.configure() call lasted 0 milliseconds. ExecutionStatus=INVOKE_NEXT_IF_ANY
  21:24:00,805 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - Trying to configure with ch.qos.logback.classic.BasicConfigurator
  21:24:00,806 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - Constructed configurator of type class ch.qos.logback.classic.BasicConfigurator
  21:24:00,806 |-INFO in ch.qos.logback.classic.BasicConfigurator@4ee1a65d - Setting up default configuration.
  21:24:00,811 |-INFO in ch.qos.logback.classic.util.ContextInitializer@481d7dd4 - ch.qos.logback.classic.BasicConfigurator.configure() call lasted 5 milliseconds. ExecutionStatus=NEUTRAL
  21:24:01,067 |-INFO in ch.qos.logback.core.joran.util.ConfigurationWatchListUtil@35a3cfe3 - Adding [jar:file:/Users/user/.gradle/caches/modules-2/files-2.1/org.springframework.boot/spring-boot/3.2.1/faa2ce019bee68a8d17529d0a08ebc427f927e13/spring-boot-3.2.1.jar!/org/springframework/boot/logging/logback/base.xml] to configuration watch list.
  21:24:01,067 |-INFO in ch.qos.logback.core.joran.spi.ConfigurationWatchList@30ba947c - URL [jar:file:/Users/user/.gradle/caches/modules-2/files-2.1/org.springframework.boot/spring-boot/3.2.1/faa2ce019bee68a8d17529d0a08ebc427f927e13/spring-boot-3.2.1.jar!/org/springframework/boot/logging/logback/base.xml] is not of type file
  21:24:01,068 |-INFO in ch.qos.logback.core.joran.util.ConfigurationWatchListUtil@35a3cfe3 - Adding [jar:file:/Users/user/.gradle/caches/modules-2/files-2.1/org.springframework.boot/spring-boot/3.2.1/faa2ce019bee68a8d17529d0a08ebc427f927e13/spring-boot-3.2.1.jar!/org/springframework/boot/logging/logback/defaults.xml] to configuration watch list.
  21:24:01,068 |-INFO in ch.qos.logback.core.joran.spi.ConfigurationWatchList@30ba947c - URL [jar:file:/Users/user/.gradle/caches/modules-2/files-2.1/org.springframework.boot/spring-boot/3.2.1/faa2ce019bee68a8d17529d0a08ebc427f927e13/spring-boot-3.2.1.jar!/org/springframework/boot/logging/logback/defaults.xml] is not of type file
  21:24:01,069 |-INFO in ch.qos.logback.core.joran.action.ConversionRuleAction - registering conversion word clr with class [org.springframework.boot.logging.logback.ColorConverter]
  21:24:01,069 |-INFO in ch.qos.logback.core.joran.action.ConversionRuleAction - registering conversion word correlationId with class [org.springframework.boot.logging.logback.CorrelationIdConverter]
  21:24:01,069 |-INFO in ch.qos.logback.core.joran.action.ConversionRuleAction - registering conversion word wex with class [org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter]
  21:24:01,069 |-INFO in ch.qos.logback.core.joran.action.ConversionRuleAction - registering conversion word wEx with class [org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter]
  21:24:01,070 |-INFO in ch.qos.logback.core.joran.util.ConfigurationWatchListUtil@35a3cfe3 - Adding [jar:file:/Users/user/.gradle/caches/modules-2/files-2.1/org.springframework.boot/spring-boot/3.2.1/faa2ce019bee68a8d17529d0a08ebc427f927e13/spring-boot-3.2.1.jar!/org/springframework/boot/logging/logback/console-appender.xml] to configuration watch list.
  21:24:01,070 |-INFO in ch.qos.logback.core.joran.spi.ConfigurationWatchList@30ba947c - URL [jar:file:/Users/user/.gradle/caches/modules-2/files-2.1/org.springframework.boot/spring-boot/3.2.1/faa2ce019bee68a8d17529d0a08ebc427f927e13/spring-boot-3.2.1.jar!/org/springframework/boot/logging/logback/console-appender.xml] is not of type file
  21:24:01,071 |-INFO in ch.qos.logback.core.joran.util.ConfigurationWatchListUtil@35a3cfe3 - Adding [jar:file:/Users/user/.gradle/caches/modules-2/files-2.1/org.springframework.boot/spring-boot/3.2.1/faa2ce019bee68a8d17529d0a08ebc427f927e13/spring-boot-3.2.1.jar!/org/springframework/boot/logging/logback/file-appender.xml] to configuration watch list.
  21:24:01,071 |-INFO in ch.qos.logback.core.joran.spi.ConfigurationWatchList@30ba947c - URL [jar:file:/Users/user/.gradle/caches/modules-2/files-2.1/org.springframework.boot/spring-boot/3.2.1/faa2ce019bee68a8d17529d0a08ebc427f927e13/spring-boot-3.2.1.jar!/org/springframework/boot/logging/logback/file-appender.xml] is not of type file
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 15
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 18
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 21
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 22
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 23
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 24
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 25
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 26
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 27
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 28
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 29
  21:24:01,073 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,074 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 36
  21:24:01,074 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,074 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 37
  21:24:01,074 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,074 |-WARN in ch.qos.logback.core.joran.action.ParamAction - <param> element is deprecated in favor of a more direct syntax.At line 38
  21:24:01,074 |-WARN in ch.qos.logback.core.joran.action.ParamAction - For details see http://logback.qos.ch/codes.html#param
  21:24:01,075 |-WARN in IfNestedWithinSecondPhaseElementSC - <if> elements cannot be nested within an <appender>, <logger> or <root> element
  21:24:01,075 |-WARN in IfNestedWithinSecondPhaseElementSC - See also http://logback.qos.ch/codes.html#nested_if_element
  21:24:01,076 |-WARN in IfNestedWithinSecondPhaseElementSC - Element <appender> at line 9 contains a nested <if> element at line 13
  21:24:01,077 |-WARN in org.springframework.boot.logging.logback.SpringProfileIfNestedWithinSecondPhaseElementSanityChecker@13eabc62 - <springProfile> elements cannot be nested within an <appender>, <logger> or <root> element
  21:24:01,077 |-WARN in org.springframework.boot.logging.logback.SpringProfileIfNestedWithinSecondPhaseElementSanityChecker@13eabc62 - Element <root> at line 94 contains a nested <springProfile> element at line 95
  21:24:01,093 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler - Setting level of logger [org.apache.catalina.startup.DigesterFactory] to ERROR
  21:24:01,093 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating ERROR level on Logger[org.apache.catalina.startup.DigesterFactory] onto the JUL framework
  21:24:01,093 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler - Setting level of logger [org.apache.catalina.util.LifecycleBase] to ERROR
  21:24:01,094 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating ERROR level on Logger[org.apache.catalina.util.LifecycleBase] onto the JUL framework
  21:24:01,094 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler - Setting level of logger [org.apache.coyote.http11.Http11NioProtocol] to WARN
  21:24:01,094 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating WARN level on Logger[org.apache.coyote.http11.Http11NioProtocol] onto the JUL framework
  21:24:01,094 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler - Setting level of logger [org.apache.sshd.common.util.SecurityUtils] to WARN
  21:24:01,094 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating WARN level on Logger[org.apache.sshd.common.util.SecurityUtils] onto the JUL framework
  21:24:01,094 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler - Setting level of logger [org.apache.tomcat.util.net.NioSelectorPool] to WARN
  21:24:01,094 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating WARN level on Logger[org.apache.tomcat.util.net.NioSelectorPool] onto the JUL framework
  21:24:01,094 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler - Setting level of logger [org.eclipse.jetty.util.component.AbstractLifeCycle] to ERROR
  21:24:01,094 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating ERROR level on Logger[org.eclipse.jetty.util.component.AbstractLifeCycle] onto the JUL framework
  21:24:01,094 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler - Setting level of logger [org.hibernate.validator.internal.util.Version] to WARN
  21:24:01,094 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating WARN level on Logger[org.hibernate.validator.internal.util.Version] onto the JUL framework
  21:24:01,094 |-INFO in ch.qos.logback.classic.model.processor.LoggerModelHandler - Setting level of logger [org.springframework.boot.actuate.endpoint.jmx] to WARN
  21:24:01,094 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating WARN level on Logger[org.springframework.boot.actuate.endpoint.jmx] onto the JUL framework
  21:24:01,094 |-INFO in ch.qos.logback.core.model.processor.AppenderModelHandler - Processing appender named [CONSOLE]
  21:24:01,094 |-INFO in ch.qos.logback.core.model.processor.AppenderModelHandler - About to instantiate appender of type [ch.qos.logback.core.ConsoleAppender]
  21:24:01,096 |-INFO in ch.qos.logback.core.model.processor.ImplicitModelHandler - Assuming default type [ch.qos.logback.classic.encoder.PatternLayoutEncoder] for [encoder] property
  21:24:01,104 |-INFO in ch.qos.logback.core.model.processor.AppenderModelHandler - Processing appender named [FILE]
  21:24:01,104 |-INFO in ch.qos.logback.core.model.processor.AppenderModelHandler - About to instantiate appender of type [ch.qos.logback.core.rolling.RollingFileAppender]
  21:24:01,106 |-INFO in ch.qos.logback.core.model.processor.ImplicitModelHandler - Assuming default type [ch.qos.logback.classic.encoder.PatternLayoutEncoder] for [encoder] property
  21:24:01,108 |-INFO in c.q.l.core.rolling.SizeAndTimeBasedRollingPolicy@2072846903 - setting totalSizeCap to 0 Bytes
  21:24:01,109 |-INFO in c.q.l.core.rolling.SizeAndTimeBasedRollingPolicy@2072846903 - Archive files will be limited to [10 MB] each.
  21:24:01,109 |-INFO in c.q.l.core.rolling.SizeAndTimeBasedRollingPolicy@2072846903 - Will use gz compression
  21:24:01,110 |-INFO in c.q.l.core.rolling.SizeAndTimeBasedRollingPolicy@2072846903 - Will use the pattern /var/folders/hx/vpf6z4md6mlg62x3rnt5h1gw0000gn/T//spring.log.%d{yyyy-MM-dd}.%i for the active file
  21:24:01,116 |-INFO in ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP@1f70504e - The date pattern is 'yyyy-MM-dd' from file name pattern '/var/folders/hx/vpf6z4md6mlg62x3rnt5h1gw0000gn/T//spring.log.%d{yyyy-MM-dd}.%i.gz'.
  21:24:01,116 |-INFO in ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP@1f70504e - Roll-over at midnight.
  21:24:01,118 |-INFO in ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP@1f70504e - Setting initial period to 2023-12-23T12:20:39.876Z
  21:24:01,123 |-INFO in ch.qos.logback.core.rolling.RollingFileAppender[FILE] - Active log file name: /var/folders/hx/vpf6z4md6mlg62x3rnt5h1gw0000gn/T//spring.log
  21:24:01,123 |-INFO in ch.qos.logback.core.rolling.RollingFileAppender[FILE] - File property is set to [/var/folders/hx/vpf6z4md6mlg62x3rnt5h1gw0000gn/T//spring.log]
  21:24:01,124 |-INFO in ch.qos.logback.classic.model.processor.RootLoggerModelHandler - Setting level of ROOT logger to INFO
  21:24:01,124 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating INFO level on Logger[ROOT] onto the JUL framework
  21:24:01,124 |-INFO in ch.qos.logback.core.model.processor.AppenderRefModelHandler - Attaching appender named [CONSOLE] to Logger[ROOT]
  21:24:01,124 |-INFO in ch.qos.logback.core.model.processor.AppenderRefModelHandler - Attaching appender named [FILE] to Logger[ROOT]
  21:24:01,124 |-INFO in ch.qos.logback.core.model.processor.AppenderModelHandler - Processing appender named [nelo]
  21:24:01,124 |-INFO in ch.qos.logback.core.model.processor.AppenderModelHandler - About to instantiate appender of type [com.naver.nelo2.logback.ThriftAppender]
  21:24:01,210 |-INFO in ch.qos.logback.core.model.processor.conditional.IfModelHandler - Condition [property("XXX").equals("yyy")] evaluated to false on line 13
  21:24:01,211 |-INFO in ch.qos.logback.core.model.processor.AppenderModelHandler - Processing appender named [nelo-async]
  21:24:01,211 |-INFO in ch.qos.logback.core.model.processor.AppenderModelHandler - About to instantiate appender of type [ch.qos.logback.classic.AsyncAppender]
  21:24:01,212 |-INFO in ch.qos.logback.core.model.processor.AppenderRefModelHandler - Attaching appender named [nelo] to ch.qos.logback.classic.AsyncAppender[nelo-async]
  21:24:01,212 |-INFO in ch.qos.logback.classic.AsyncAppender[nelo-async] - Attaching appender named [nelo] to AsyncAppender.
  21:24:01,212 |-INFO in ch.qos.logback.classic.AsyncAppender[nelo-async] - Setting discardingThreshold to 409
  21:24:01,212 |-INFO in ch.qos.logback.classic.model.processor.RootLoggerModelHandler - Setting level of ROOT logger to DEBUG
  21:24:01,212 |-INFO in ch.qos.logback.classic.jul.LevelChangePropagator@507a330c - Propagating DEBUG level on Logger[ROOT] onto the JUL framework
  21:24:01,212 |-INFO in ch.qos.logback.core.model.processor.DefaultProcessor@59bff66e - End of configuration.
  21:24:01,212 |-INFO in org.springframework.boot.logging.logback.SpringBootJoranConfigurator@585c5c06 - Registering current configuration as safe fallback point
  ```

#### `<param>` deprecated

- 에러 메시지 중 `<param> element is deprecated in favor of a more direct syntax.At line 15`
- https://logback.qos.ch/codes.html#param
- logback-spring.xml 파일에서 `<param>` 대신에 파라미터 이름을 `<camelCase>` 태그로 사용하도록 변경

#### `<springProfile>` 중첩 금지

- 에러 메시지 중 `<springProfile> elements cannot be nested within an <appender>, <logger> or <root> element`
- `<springProfile>`를 `<root>` 밖으로 빼고 `<springProfile>` 안에 `<root>`를 포함하도록 변경

#### `<if>` 중첩 금지

- 에러 메시지 중 `<if> elements cannot be nested within an <appender>, <logger> or <root> element`
- https://logback.qos.ch/codes.html#nested_if_element
- `<if>`를 `<appender>, <logger> or <root>` 밖으로 빼고 `<variable>`을 사용해서 변수 선언

### Elasticsearch 버전 이슈

- [Spring Boot 3 에서 Elasticsearch 7 사용](https://github.com/HomoEfficio/dev-tips/blob/master/Spring-Boot-3-Elasticsearch-7.md)
