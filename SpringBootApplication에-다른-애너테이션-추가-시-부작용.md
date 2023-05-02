# `@SpringBootApplication`에 다른 애너테이션 추가 시 부작용

다음과 같이 `@SpringBootApplication`에 다른 애너테이션도 붙여서 사용해왔는데,

```java
@EnableJpaAuditing
@SpringBootApplication
public class MonolithSimpleMallApplication {

    public static void main(String[] args) {
        SpringApplication.run(MonolithSimpleMallApplication.class, args);
    }

}

```

이 방식은 다음과 같이 `@WebMvcTest`로 일부 컨트롤러만 테스트 할 때,

```java
@WebMvcTest(DummyHomeController.class)
public class BasicAuthTest {
...
```

다음과 같은 에러를 유발한다.

```java
  ...
Caused by: java.lang.IllegalArgumentException: JPA metamodel must not be empty!
  at org.springframework.util.Assert.notEmpty(Assert.java:464)
  at org.springframework.data.jpa.mapping.JpaMetamodelMappingContext.<init>(JpaMetamodelMappingContext.java:58)
  at org.springframework.data.jpa.repository.config.JpaMetamodelMappingContextFactoryBean.createInstance(JpaMetamodelMappingContextFactoryBean.java:80)
  at org.springframework.data.jpa.repository.config.JpaMetamodelMappingContextFactoryBean.createInstance(JpaMetamodelMappingContextFactoryBean.java:44)
  at org.springframework.beans.factory.config.AbstractFactoryBean.afterPropertiesSet(AbstractFactoryBean.java:142)
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1855)
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1792)
  ... 92 more
```

이유는 `@WebMvcTest`로 일부 컨트롤러만 포함해서 테스트를 실행할 때 `@SpringBootApplication`가 붙은 메인 클래스를 찾아 실행하는 데 여기에 `@EnableJpaAuditing`가 붙어서 있어서 JPA Auditing 을 활성화하려고 하지만, `@WebMvcTest`가 실행하는 테스트 용 애플리케이션 컨텍스트는 [레퍼런스 문서](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-testing-spring-boot-applications-testing-autoconfigured-mvc-tests)에 있는 것처럼 일반적인 `@Component` 붙은 클래스를 빈으로 등록하지 않는다. 따라서 JPA metamodel 이 비어있게 되고 결국 위와 같은 에러를 발생시킨다.

해결 방법은 간단하다.

`@SpringBootApplication` 외에 다른 애너테이션은 별도의 `@Configuration` 클래스를 만들고 그 클래스에 해당 애너테이션을 붙여주면 된다. 예를 들어 `@EnableJpaAuditing`은 다음과 같이 새 클래스에 붙여주면 된다.

```java
@EnableJpaAuditing
@Configuration
public class JpaConfig {
}

```
