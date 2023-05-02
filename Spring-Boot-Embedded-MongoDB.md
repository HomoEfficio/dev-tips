# Spring Boot Embedded MongoDB

꼭 테스트 용도가 아니라 학습용으로 간편한 임베디드 몽고디비를 사용하고 싶다.

## 설정

### dependency

testImplementation 말고 아래와 같이 implementation 사용

```
    implementation("de.flapdoodle.embed:de.flapdoodle.embed.mongo")
```

### application.yml

위와 같이 dependency 설정하고 애플리케이션 실행하면 아래와 같이 버전이 필요하다는 예외 발생

```
Caused by: java.lang.IllegalStateException: Set the spring.mongodb.embedded.version property or define your own MongodConfig bean to use embedded MongoDB
    at org.springframework.util.Assert.state(Assert.java:76) ~[spring-core-5.3.21.jar:5.3.21]
    at org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration.determineVersion(EmbeddedMongoAutoConfiguration.java:146) ~[spring-boot-autoconfigure-2.7.1.jar:2.7.1]
    at org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration.embeddedMongoConfiguration(EmbeddedMongoAutoConfiguration.java:126) ~[spring-boot-autoconfigure-2.7.1.jar:2.7.1]
    at java.base/jdk.internal.reflect.DirectMethodHandleAccessor.invoke(DirectMethodHandleAccessor.java:104) ~[na:na]
    at java.base/java.lang.reflect.Method.invoke(Method.java:577) ~[na:na]
    at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:154) ~[spring-beans-5.3.21.jar:5.3.21]
    ... 155 common frames omitted
```

아래와 같이 버전을 명시해준다. 버전은 External Libraries 폴더에서 확인 가능.

```yml
spring:
  mongodb:
    embedded:
      version: 3.4.6

```

다시 실행하면 아래와 같이 기동 성공하긴 하는데 이런 예외가..

```
java.lang.NoSuchFieldException: handle
    at java.base/java.lang.Class.getDeclaredField(Class.java:2642)
    at de.flapdoodle.embed.process.runtime.Processes.windowsProcessId(Processes.java:112)
    at de.flapdoodle.embed.process.runtime.Processes.access$200(Processes.java:48)
    at de.flapdoodle.embed.process.runtime.Processes$PidHelper$3.getPid(Processes.java:218)
```

검색해보니 기동에는 문제가 없는데 여차저차 로그가 찍히는 불편함뿐이라는..

https://github.com/flapdoodle-oss/de.flapdoodle.embed.process/pull/123#issuecomment-859476883

어차피 학습용이므로 기동만 되면 뭐..




