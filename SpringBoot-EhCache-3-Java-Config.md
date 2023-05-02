# Spring Boot EhCache 3 Java Config

EhCache 는 보통 ehcache.xml 파일로 설정을 하는데 Java Config 방식으로도 가능하다.

`javax.cache.CacheManager` 빈을 등록해야 캐시가 제대로 동작한다.

`CacheConfigurationBuilder.newCacheConfigurationBuilder()`를 사용해서 EhCacheConfigurationBuilder 를 생성하고,  
`Eh107Configuration.fromEhcacheCacheConfiguration()`를 사용해서 `javax.cache.configuration.Configuration`을 만들어내고,  
`javax.cache.CacheManager.createCache()`를 사용해서 캐시를 생성한다.


```kotlin
package aa.bb.cc.dd

import aa.bb.cc.dd.dto.XXXOut
import org.ehcache.config.builders.*
import org.ehcache.config.units.MemoryUnit
import org.ehcache.event.CacheEvent
import org.ehcache.event.CacheEventListener
import org.ehcache.event.EventType
import org.ehcache.jsr107.Eh107Configuration
import org.slf4j.LoggerFactory
import org.springframework.cache.annotation.EnableCaching
import org.springframework.cache.interceptor.SimpleKey
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.lang.Long
import java.time.Duration
import javax.cache.CacheManager
import javax.cache.Caching

@Configuration
@EnableCaching
class CacheConfig {

    @Bean
    fun ehCacheManager(): CacheManager {
        val cachingProvider = Caching.getCachingProvider()
        log.info("OOO CacheConfig, cachingProvider: ${cachingProvider}")

        val cacheManager: CacheManager = cachingProvider.cacheManager
        log.info("OOO CacheConfig, cacheManager: $cacheManager")

        log.info("OOO CacheConfig, cacheManager.properties: ${cacheManager.properties}")


        val cacheEventListenerConfigurationBuilder = CacheEventListenerConfigurationBuilder.newEventListenerConfiguration(
            CacheLogger::class.java,
            setOf(
                EventType.CREATED,
                EventType.UPDATED,
                EventType.EXPIRED,
                EventType.REMOVED,
                EventType.EVICTED,
            )
        )

        cacheManager.createCache(
            CacheNames.category1,
            Eh107Configuration.fromEhcacheCacheConfiguration(
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    SimpleKey::class.java,
                    ArrayList::class.java,
                    ResourcePoolsBuilder.heap(1).offheap(1, MemoryUnit.MB)
                ).withExpiry(
                    ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofMinutes(10))
                ).withService(
                    cacheEventListenerConfigurationBuilder
                )
            ),
        )

        cacheManager.createCache(
            CacheNames.category2,
            Eh107Configuration.fromEhcacheCacheConfiguration(
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    Long::class.java,
                    ArrayList::class.java,
                    ResourcePoolsBuilder.heap(100).offheap(1, MemoryUnit.MB)
                ).withExpiry(
                    ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofMinutes(10))
                ).withService(
                    cacheEventListenerConfigurationBuilder
                )
            ),
        )

        cacheManager.createCache(
            CacheNames.zccPublishedPackageInfo,
            Eh107Configuration.fromEhcacheCacheConfiguration(
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    String::class.java,
                    XXXOut::class.java,
                    ResourcePoolsBuilder.heap(100).offheap(1, MemoryUnit.MB)
                ).withExpiry(
                    ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofDays(1))
                ).withService(
                    cacheEventListenerConfigurationBuilder
                )
            ),
        )


        return cacheManager
    }

    private val log = LoggerFactory.getLogger(javaClass)
}

class CacheLogger : CacheEventListener<Any, Any> {

    private val log = LoggerFactory.getLogger(javaClass)

    override fun onEvent(cacheEvent: CacheEvent<out Any, out Any>) {
        log.info("Key: [${cacheEvent.key}] | EventType: [${cacheEvent.type}] | Old value: [${cacheEvent.oldValue}] | New value: [${cacheEvent.newValue}]")
    }
}

object CacheNames {
    const val category1 = "category1"
    const val category2 = "category2"
    const val xxx = "xxx"
}

```

캐시 사용하는 쪽은 다음과 같다.

```kotlin
    // CacheManager 빈을 주입받는다

    /**
     * 호출하면 캐시값이 있으면 캐시값을 반환하고 메서드 내부를 실행하지 않는다.
     */
    @Cacheable(cacheNames = [CacheNames.zccPublishedPackageInfo], key = "#zccId")
    fun getPublishedZccPackageInfo(zccId: String): ZccPackageInfoForGameSystemOut {
        val dbZcc = try {
            zccRepository.findByZccId(zccId)
        } catch (e: Exception) {
            return ZccPackageInfoForGameSystemOut(
                isSuccess = false,
                message = "ZCC not found - zccId: {$zccId}",
            )
        }

        if (!dbZcc.hasPublishedVersion()) {
            return ZccPackageInfoForGameSystemOut(
                isSuccess = false,
                message = "zccId [$zccId] 인 ZCC는 출시 버전이 없습니다.",
            )
        }

        val publishedVersion = dbZcc.getPublishedVersion()!!
        return ZccPackageInfoForGameSystemOut(
            isSuccess = true,
            message = null,
            worldName = publishedVersion.getName(),
            packageInfo = publishedVersion.getPackageInfo()
        )
    }


    /**
     * 호출되면 메서드 내부가 항상 실행되며 캐시 아이템을 삭제하거나 새 값으로 업데이트한다.
     * 캐시 아이템 삭제 후 null 반환 시 아래와 같이 type 관련 에러가 발생하므로,
     *   java.lang.ClassCastException: Invalid value type, expected : me.zepeto.studio.world.admin.query.transfer.dto.gamesystem.ZccPackageInfoForGameSystemOut but was : org.springframework.cache.support.NullValue
     *       at org.ehcache.impl.store.BaseStore.checkValue(BaseStore.java:73)
     * 아래 메서드를 호출하는 곳에 아래와 같이 try-catch 로 타입 에러를 잡아서 무시한다.
     *  try {
     *      zccQuery.updateCacheXxx(xxxOid)
     *  } catch (e: ClassCastException) {
     *      // 캐시 아이템 삭제 후 null 이 되면서 발생하는 아래 에러 무시
     *      // java.lang.ClassCastException: Invalid value type, expected : me.zepeto.studio.world.admin.query.transfer.dto.gamesystem.ZccPackageInfoForGameSystemOut but was : org.springframework.cache.support.NullValue
     *      // at org.ehcache.impl.store.BaseStore.checkValue(BaseStore.java:73)
     *      log.info("Removed xxx cache item with xxxOid {$xxxOid}")
     *  }
     */
    @CachePut(cacheNames = [CacheNames.xxx], key = "#xxxOid")
    fun updateCacheXxx(xxxOid: String): XxxOut? {
        val dbXxx = try {
            xxxRepository.findByXxxOid(xxxOid)
        } catch (e: Throwable) {
            log.info("Removing xxx cache item with xxxOid {$xxxOid} because of not found xxx")
            removeCacheItem(xxxOid)
            return null
        }

        val publishedVersion = dbXxx.getPublishedVersion()
        if (publishedVersion == null) {
            log.info("Removing xxx cache item with xxxOid {$xxxOid} because of not found published xxx version")
            removeCacheItem(zccId)
            return null
        }

        return XxxOut(
            isSuccess = true,
            message = null,
            name = publishedVersion.getName(),
            info = publishedVersion.getXxxInfo(),
        )
    }
    
    private fun removeCacheItem(zccId: String) {
        cacheManager.getCache(
            CacheNames.xxx,
            String::class.java, XxxOut::class.java
        ).remove(zccId)
    }
```
