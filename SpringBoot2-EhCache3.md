# Spring Boot 2 + EhCache3

스프링 부트와 널리 사용되는 캐시 솔루션인 EhCache3 설정 방법을 알아본다.

스프링 부트2 의 autoconfiguration 은 EhCache3가 아니라 EhCache2 가 기본값인 것 같다.  
스프링 부트 2.5.1 기준으로 실행 로그를 보면 아래와 같이 `net.sf.ehcache.*` 클래스를 찾는 것을 확인할 수 있다.

```
...
   CacheMeterBinderProvidersConfiguration.EhCache2CacheMeterBinderProviderConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'net.sf.ehcache.Ehcache' (OnClassCondition)
...
   EhCacheCacheConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'net.sf.ehcache.Cache' (OnClassCondition)
...
```


## 의존관계 설정

그래서 EhCache3를 사용하려면 아래와 같이 3개의 라이브러리를 모두 추가해야 한다. EhCache3는 JSR-107 스펙을 구현하고 있어서 javax.cache 가 필요하다. 결국 아래 3가지 중 하나라도 없으면 EhCache3 기반 캐시는 제대로 동작하지 않는다.

```kotlin
    implementation("org.springframework.boot:spring-boot-starter-cache")
    implementation("org.ehcache:ehcache:3.9.5")
    implementation("javax.cache:cache-api:1.1.1")
```


## application.yml

ehcache3 을 사용하려면 아래와 같이 `jcache` 로 지정해줘야 한다.

```yml
spring:
  cache:
    jcache:
      config: classpath:ehcache/ehcache.xml

```

아래와 같이 `ehcache`로 지정하는 건 EhCache2 용인 것 같다.


```yml
spring:
  cache:
    ehcache:
      config: classpath:ehcache/ehcache.xml

```


## ehcache.xml

```xml
<config
        xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
        xmlns='http://www.ehcache.org/v3'
        xmlns:jsr107='http://www.ehcache.org/v3/jsr107'>
<!--    <persistence directory="spring-boot-ehcache/cache" />-->
    <cache-template name="default">
        <listeners>
            <listener>
                <class>aaa.bbb.CacheEventListenerImpl</class>
                <event-firing-mode>ASYNCHRONOUS</event-firing-mode>
                <event-ordering-mode>UNORDERED</event-ordering-mode>
                <events-to-fire-on>CREATED</events-to-fire-on>
                <events-to-fire-on>UPDATED</events-to-fire-on>
                <events-to-fire-on>EXPIRED</events-to-fire-on>
                <events-to-fire-on>REMOVED</events-to-fire-on>
                <events-to-fire-on>EVICTED</events-to-fire-on>
            </listener>
        </listeners>
        <resources>
            <heap>100</heap>
            <offheap unit="MB">3</offheap>
<!--            <disk persistent="true" unit="MB">20</disk>-->
        </resources>
    </cache-template>

    <cache alias="category1" uses-template="default">
        <value-type>java.util.ArrayList</value-type>
        <expiry>
            <ttl unit="seconds">3</ttl>
        </expiry>
    </cache>

    <cache alias="category2" uses-template="default">
        <key-type>java.lang.Long</key-type>
        <value-type>java.util.ArrayList</value-type>
        <expiry>
            <ttl unit="seconds">30</ttl>
        </expiry>
    </cache>
</config>

```

ehcache 설정 파일 상세 내용은 [레퍼런스 문서](https://www.ehcache.org/documentation/3.8/107.html)를 참고하고 여기에서는 간단하게 살펴본다.


### `<persistence>`

주석처리한 `<persistence>`는 어딘가 예제에 지정돼 있어서 마치 반드시 있어야 하는 것처럼 돼 있는데, 지속성 있는 Persistent Cache 를 사용할 때만 필요하고 그냥 휘발성 메모리 캐시를 사용할 때는 지정하지 않아도 된다.


### `<cache-template>`

`<cache-template>`는 여러 캐시에서 상속받아서 사용할 수 있는 공통 내용을 설정한다.


### `<listeners>`

`<listeners>`도 필수는 아니다. 그 안에 포함된 것들 역시 필수가 아니고 그냥 설명을 위해 캐시 관련 모든 이벤트를 기술해둔 건데, 처음에 캐시 동작 파악할 때만 활성화 시켜뒀다가 나중에는 필요한 이벤트만 남겨두면 된다.

리스너 클래스는 `org.ehcache.event.CacheEventListener` 인터페이스를 구현한 클래스를 지정해주면 되며, 대략 아래와 같은 예제를 쉽게 찾을 수 있다.

```kotlin
class CacheLogger : CacheEventListener<Any, Any> {

    private val log = LoggerFactory.getLogger(javaClass)

    override fun onEvent(cacheEvent: CacheEvent<out Any, out Any>) {
        // error 가 아니지만 터미널에 빨간색으로 눈에 띄라고 error 사용
        log.error("Key: [${cacheEvent.key}] | EventType: [${cacheEvent.type}] | Old value: [${cacheEvent.oldValue}] | New value: [${cacheEvent.newValue}]")
    }
}

```

### `<resources>`

캐시 저장에 사용할 자원을 설정한다. `<heap>`, `<offheap>`, `<disk>`는 ehcache 에서 storage tier 라는 개념으로 정의한 항목들이다. 쉽게 말해 레지스터, L1, L2, ... 캐시가 있는 것과 비슷한 개념이다.  
자세한 내용은 [EhCache 문서](https://www.ehcache.org/documentation/3.8/caching-concepts.html#storage-tiers) 를 참고하고 간단하게 정리하면 다음과 같다.

- heap: JVM heap 메모리를 의미하며 GC 대상이고, 접근 속도가 가장 빠르다.
- offheap: JVM 외부의 메모리이며 사용할 때 결국 heap을 거치게 되므로 heap 보다는 느리지만 더 많은 용량을 사용할 수 있다.
- disk: 디스크이므로 속도는 느리고 용량은 크다. disk 를 사용하려면 접근 속도가 빠른 디스크를 사용하는 것이 좋다.
- 1개의 tier 로 구성하면 해당 tier에만 데이터가 캐시된다.
- 2개 이상의 tier 로 구성하면 자주 조회되는 데이터는 near cache(2개 이상 중에서 빠른 tier들)에 저장되고 자주 조회되지 않는 데이터는 authority cache(2개 이상 중에서 가장 느린 1개의 tier)에 저장된다.

### `<cache>`

`<cache>`에서 실제 사용할 캐시를 설정한다. `alias`에 지정한 이름과 나중에 `@Cacheable(cacheNames = [])`에 지정한 이름이 동일해야 한다.

`<key-type>`도 필수는 아니다. 지정해주지 않으면 스프링의 [디폴트 키 생성 전략](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache-annotations-cacheable-default-key)에 따라 키가 지정된다.


### `<expiry>`

말 그대로 캐시의 만료 시점을 의미한다. 역시 필수는 아니며 지정하지 않으면 만료되지 않는, 그러니까 애플리케이션 메모리에 계속 남아 있는 캐시가 만들어진다.


## 캐시 적용

적용은 간단하다. [스프링에서 제공하는 애노테이션 또는 JSR-107 애노테이션](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache-jsr-107-summary) 을 캐시를 적용할 곳에 붙여주면 된다.

아래는 가장 널리 사용되는 방식인 메서드에 붙여주는 사례다.

```kotlin
@Service
@Transactional(readOnly = true)
class CategoryQuery(
    private val categoryRepository: CategoryRepository,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Cacheable(cacheNames = ["category1"])
    fun findFirstCategories(): List<CategoryOut> {
        log.error("DB 조회 - category1")
        return categoryRepository.findAllByParentIdNull()
            .map { CategoryOut.fromEntity(it) }
    }

    @Cacheable(cacheNames = ["category2"], key = "#parentId")
    fun findSecondCategoriesByFirstCategory(parentId: Long): List<CategoryOut> {
        log.error("DB 조회 - category2")
        return categoryRepository.findAllByParentId(parentId)
            .map { CategoryOut.fromEntity(it) }
    }
}

```

1차 카테고리 목록을 조회하는 메서드는 파라미터가 없다. 저장되는 항목이 1차 카테고리 전체 목록 1개 뿐이므로 굳이 key가 필요하지는 않다. [스프링 문서](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache-annotations-cacheable-default-key)에 보면 key를 지정해주지 않으면 `SimpleKey.EMPTY`가 key로 사용된다고 써 있다.

따라서 이처럼 캐시에 저장되는 항목이 1개(1차 카테고리 전체 목록 1개 - 물론 목록 내 원소는 여러개지만 목록은 1개)뿐이라면 key를 지정하지 않아도 `SimpleKey.EMPTY`를 key로 해서 저장되고, 이후 조회 요청이 오면 DB를 조회하지 않고 캐시에서 `SimpleKey.EMPTY`를 key로 값을 조회해서 바로 반환한다. 이는 앞서 작성한 `CacheEventListener` 구현체를 통해 찍히는 로그로 확인할 수 있다.

2차 카테고리는 1차 카테고리에 종속되는 부속 카테고리로서 1차 카테고리가 먼저 결정돼야 확인할 수 있다. 따라서 1차 카테고리의 식별자를 key로 지정한다.

캐시에 저장되는 값이 카테고리 목록이므로 ehcache.xml 파일에서 `value-type`을 `java.util.ArrayList`로 지정했다. 이 목록에 들어가는 클래스는 `CategoryOut`인데, 캐시에 저장되려면 반드시 `Serializable` 마커 인터페이스를 붙여줘야만 Ser돼서 캐시에 저장되고, Deser돼서 캐시에서 꺼내올 수 있다.

`@Cacheable` 외에도 여러 애노테이션이 있으며 [스프링 문서](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache-jsr-107-summary)를 참고한다.


## 실행 로그

대략 아래와 같은 로그와 함께 ehcache.xml 에서 설정한 내용대로 로그가 찍히면 EhCache3 캐시가 제대로 생성된 것이다.

```
[  restartedMain] org.ehcache.core.spi.ServiceLocator      : Starting 18 Services...
[  restartedMain] o.e.c.i.s.DefaultStatisticsService       : Starting service
[  restartedMain] org.ehcache.core.spi.ServiceLocator      : All Services successfully started, 18 Services in 6ms
...
```

실제 캐시 저장, 만료 등은 앞서 작성한 `CacheEventListener` 구현체에서 찍어주는 로그로 확인할 수 있다.

로그를 확인해 보면 `<expiry>`에 지정한 만료 시점에 도달했을 때 캐시 항목이 자동으로 지워지지는 않고, 만료 시점 이후에 처음 캐시에 접근할 때 만료된 캐시 항목을 지우고 바로 이어서 DB 조회해서 값을 가져온 후 다시 캐시에 담는 다는 것을 확인할 수 있다.

