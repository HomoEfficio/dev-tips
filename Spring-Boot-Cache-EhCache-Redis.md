# Spring Boot Cache로 EhCache와 Redis를 함께 사용하기

스프링이 제공하는 캐시 추상화를 활용해서,  
로컬이나 dev 환경에서는 EhCache를 사용하고,  
prod 환경에서는 Redis를 사용하도록 설정

### 주요 포인트

- 서로 다른 캐시 구현체에 의해 구성되는 서로다른 CacheManager를 Spring CacheManager로 추상화하는 과정
- EhCache 설정을 ehcache.xml 이 아닌 코틀린 코드로 설정하는 과정
- Redis에서 Collection도 Deserialization 할 수 있도록 설정하는 과정
- `@Profile` 대신에 `@ConditionalOnProperty`나 `@ConditionalOnExpression`을 사용해서 환경에 맞는 캐시를 사용하도록 설정하는 과정

### 의존 관계

```kotlin
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.springframework.boot:spring-boot-starter-cache")
    implementation("org.ehcache:ehcache:3.9.5")
    implementation("javax.cache:cache-api:1.1.1")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
```

### 설정 클래스

```kotlin
package aa.bb.cc.config

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.type.CollectionType
import com.fasterxml.jackson.databind.type.TypeFactory
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import aa.bb.cc.dd.XxxOut
import aa.bb.cc.dd.YyyOut
import org.ehcache.config.builders.CacheConfigurationBuilder
import org.ehcache.config.builders.CacheEventListenerConfigurationBuilder
import org.ehcache.config.builders.ExpiryPolicyBuilder
import org.ehcache.config.builders.ResourcePoolsBuilder
import org.ehcache.config.units.MemoryUnit
import org.ehcache.event.CacheEvent
import org.ehcache.event.CacheEventListener
import org.ehcache.event.EventType
import org.ehcache.jsr107.Eh107Configuration
import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Value
import org.springframework.boot.autoconfigure.condition.ConditionalOnExpression
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.cache.CacheManager
import org.springframework.cache.annotation.EnableCaching
import org.springframework.cache.interceptor.SimpleKey
import org.springframework.cache.jcache.JCacheCacheManager
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.redis.cache.RedisCacheConfiguration
import org.springframework.data.redis.cache.RedisCacheManager.RedisCacheManagerBuilder
import org.springframework.data.redis.connection.RedisConnectionFactory
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer
import org.springframework.data.redis.serializer.RedisSerializationContext
import java.time.Duration
import javax.cache.Caching


@Configuration
@EnableCaching
class CacheConfig {

    private val log = LoggerFactory.getLogger(javaClass)

    @Bean
    @ConditionalOnExpression("#{ \${cache.ehcache} != true }")
    fun redisCacheManager(
        redisConnectionFactory: RedisConnectionFactory,
        @Value("\${xxYyZz}") xxYyZz: String,
    ): CacheManager {
        val jacksonObjectMapper = jacksonObjectMapper()

        val yyySerializer = yyySerializer(jacksonObjectMapper)

        return RedisCacheManagerBuilder.fromConnectionFactory(redisConnectionFactory)
            .withCacheConfiguration(
                CacheNames.yyy1,
                RedisCacheConfiguration.defaultCacheConfig()
                    .prefixCacheNameWith(redisMmmPrefix(xxYyZz))
                    .serializeValuesWith(
                        RedisSerializationContext.SerializationPair
                            .fromSerializer(
                                yyySerializer
                            )
                    )
                    .entryTtl(Duration.ofMinutes(10))
            )
            .withCacheConfiguration(
                CacheNames.yyy2,
                RedisCacheConfiguration.defaultCacheConfig()
                    .prefixCacheNameWith(redisMmmPrefix(xxYyZz))
                    .serializeValuesWith(
                        RedisSerializationContext.SerializationPair
                            .fromSerializer(
                                yyySerializer
                            )
                    )
                    .entryTtl(Duration.ofMinutes(10))
            )
            .withCacheConfiguration(
                CacheNames.xxx,
                RedisCacheConfiguration.defaultCacheConfig()
                    .prefixCacheNameWith(redisMmmPrefix(xxYyZz))
                    .serializeValuesWith(
                        RedisSerializationContext.SerializationPair
                            .fromSerializer(
                                xxxSerializer(jacksonObjectMapper)
                            )
                    )
                    .entryTtl(Duration.ofDays(1))
            ).build()
    }

    private fun redisMmmPrefix(xxYyZz: String) = "${CacheNames.prefix}-$xxYyZz-"

    private fun xxxSerializer(jacksonObjectMapper: ObjectMapper): Jackson2JsonRedisSerializer<XxxOut> {
        val xxxSerializer = Jackson2JsonRedisSerializer(XxxOut::class.java)
        xxxSerializer.setObjectMapper(jacksonObjectMapper)
        return xxxSerializer
    }

    private fun yyySerializer(jacksonObjectMapper: ObjectMapper): Jackson2JsonRedisSerializer<List<YyyOut>> {
        val typeListOfYyyOut: CollectionType =
            TypeFactory.defaultInstance().constructCollectionType(List::class.java, YyyOut::class.java)
        val yyySerializer = Jackson2JsonRedisSerializer<List<YyyOut>>(typeListOfYyyOut)
        yyySerializer.setObjectMapper(jacksonObjectMapper)
        return yyySerializer
    }


    @Bean
    @ConditionalOnProperty(name = ["cache.ehcache"], havingValue = "true")
    fun ehCacheManager(): CacheManager {
        val cachingProvider = Caching.getCachingProvider()
        log.info("OOO CacheConfig, cachingProvider: ${cachingProvider}")

        val cacheManager: javax.cache.CacheManager = cachingProvider.cacheManager
        log.info("OOO CacheConfig, cacheManager: $cacheManager")

        log.info("OOO CacheConfig, cacheManager.properties: ${cacheManager.properties}")


        val cacheEventListenerConfigurationBuilder =
            CacheEventListenerConfigurationBuilder.newEventListenerConfiguration(
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
            CacheNames.yyy1,
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
            CacheNames.yyy2,
            Eh107Configuration.fromEhcacheCacheConfiguration(
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    java.lang.Long::class.java,
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
            CacheNames.xxx,
            Eh107Configuration.fromEhcacheCacheConfiguration(
                CacheConfigurationBuilder.newCacheConfigurationBuilder(
                    String::class.java,
                    XxxOut::class.java,
                    ResourcePoolsBuilder.heap(100).offheap(1, MemoryUnit.MB)
                ).withExpiry(
                    ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofDays(1))
                ).withService(
                    cacheEventListenerConfigurationBuilder
                )
            ),
        )

        return JCacheCacheManager(cacheManager)
    }

    class CacheLogger : CacheEventListener<Any, Any> {

        private val log = LoggerFactory.getLogger(javaClass)

        override fun onEvent(cacheEvent: CacheEvent<out Any, out Any>) {
            log.info("Key: [${cacheEvent.key}] | EventType: [${cacheEvent.type}] | Old value: [${cacheEvent.oldValue}] | New value: [${cacheEvent.newValue}]")
        }
    }
}

object CacheNames {
    const val prefix = "omw"
    const val yyy1 = "yyy1"
    const val yyy2 = "yyy2"
    const val xxx = "xxx"
}

```


