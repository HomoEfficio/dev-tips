# Spring Boot Unit Test에서 YAML Property Loading

application.yml 이 다음과 같이 작성돼 있고,

```yaml
aa.bb.cc: abc

aa.bb:
  ddd: abd
```

아래와 같은 Properties 클래스가 있고,

```kotlin
@ConfigurationProperties(prefix = "aa.bb")
@ConstructorBinding
data class HelloProperties (
    val cc: String,
    val ddd: String
)
```

아래와 같은 Main 클래스가 있을 때,

```kotlin
@SpringBootApplication
@EnableConfigurationProperties(HelloProperties::class)
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args)
}
```

애플리케이션을 실행하면 HelloProperties 빈이 생기고 그 안의 `cc`에는 `abc`, `ddd`에는 `abd`가 잘 로딩된다.

그런데 아래와 같이 `@SpringBootTest`를 사용하지 않고 필요한 것만 로딩하는 테스트 코드에서는, HelloProperties 빈이 만들어지기는 하는데 `ddd`에 빈 문자열 값이 들어오거나, 아니면 `ddd`를 인식하지 못해서 HelloProperties 빈이 아에 생성되지 못한다. 

```kotlin
@ExtendWith(SpringExtension::class)
@EnableConfigurationProperties(HelloProperties::class)
@TestPropertySource("classpath:application.yml")
internal class HelloPropertiesTest {

    @Autowired
    private lateinit var helloProps: HelloProperties


    @Test
    internal fun showProps() {
        println("single-line: " + helloProps.cc)
        println("multi-line: " + helloProps.ddd)
    }
}
```

하지만 `@SpringBootTest`를 사용하면 제대로 로딩된다.

또는 yml 파일에서 ddd 까지도 모두 한 줄에 그러니까 aa.bb.ddd 라고 정의하면 `@SpringBootTest` 없이도 제대로 로딩된다.

왜 이럴까?

이유는 **테스트에서는 기본적으로 YAML 파일을 로딩하는 기능이 포함되지 않기 때문이다.**

그렇다면 `@SpringBootTest`에서는 어떻게 YAML 파일을 제대로 로딩하는 걸까?

이유는 `@SpringBootTest`에는 YAML 파일 로딩 기능이 포함된 `@BootstrapWith(SpringBootTestContextBootstrapper.class)`가 포함돼 있기 때문이다.

그런데 `@BootstrapWith(SpringBootTestContextBootstrapper.class)`를 사용하면 테스트에 필요한 요소들뿐 아니라 애플리케이션 전체 컨텍스트를 로딩하기 때문에 테스트 시간이 오래 걸린다.

그렇다면 `@BootstrapWith(SpringBootTestContextBootstrapper.class)`를 사용하지 않고 YAML을 제대로 로딩하려면 어떻게 해야할까?



## PropertySourceFactory

검색해보면 `ConfigFileApplicationContextInitializer`, `PropertySourcesPlaceholderConfigurer`, `YamlPropertySourceLoader`, `ConfigurationPropertiesBindingPostProcessor` 등 여러 방법이 나오는데 또 다른 빈이 있어야 제대로 동작하는 것 같다.

아래와 같이 `PropertySourceFactory`를 사용하면 애초에 의도한 것처럼 다른 빈 없이도 최소한의 설정으로 YAML property를 잘 로딩할 수 있다. 아래와 같은 구현체를 만들어서,

```kotlin
class YamlPropertySourceFactory : PropertySourceFactory {

    override fun createPropertySource(name: String?, encodedResource: EncodedResource): PropertySource<*> {

        encodedResource.resource.filename?.let {
            val factory = YamlPropertiesFactoryBean()
            factory.setResources(encodedResource.resource)
            factory.afterPropertiesSet()

            val properties: Properties? = factory.getObject()

            return PropertiesPropertySource(it, properties!!)
        }

        throw IllegalArgumentException("Resource file name is null or empty.")
    }
}
```

테스트 코드에서 `@TestPropertySource` 대신에 `@PropertySource`를 사용하고 `factory`를 지정해주면, `@BootstrapWith(SpringBootTestContextBootstrapper.class)` 없이도 YAML property도 정상적으로 로딩할 수 있다.

```
@ExtendWith(SpringExtension::class)
@EnableConfigurationProperties(HelloProperties::class)
@Import(TestConfig::class)
@PropertySource("classpath:application.yml", factory = YamlPropertySourceFactory::class)  // 여기!!
internal class HelloPropertiesTest {

    @Autowired
    private lateinit var helloProps: HelloProperties


    @Test
    internal fun showProps() {
        println("single-line: " + helloProps.cc)
        println("multi-line: " + helloProps.ddd)
    }
}
```

