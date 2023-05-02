# Spring Test YamlPropertySourceFactory

test용 yml을 따로 지정해서 사용할 수도 있지만, 개발이 진행되면서 원래 yml에만 반영되고 test용 yml에는 반영되지 않아 테스트가 의도대로 동작하지 않게 되는 단점이 있다.

어차피 둘(원래 yml, test yml)의 동기화는 필요한데 동기화가 현실적으로 잘 안 된다면 그냥 test에서도 원래 yml을 읽어서 사용하도록 구성해보자.

## 큰 흐름

- 테스트 코드에서 다음과 같이 `@PropertySouerce`를 사용해서 yml 파일 위치와 팩토리 클래스를 지정한다.

```kotlin
@PropertySource("classpath:/config/application.yml", factory = YamlPropertySourceFactory::class)
```

- 위와 같이 하면 yml 파일에 있는 내용을 `YamlPropertySourceFactory` 가 읽어서 테스트 컨텍스트의 `PropertySource`에 넣어주고, 테스트 코드에서는 테스트 컨텍스트에 들어 있는 property 값을 사용할 수 있다.

## YamlPropertySourceFactory

`PropertySourceFactory` 인터페이스를 구현한 커스텀 팩토리 클래스다. 다음과 같이 작성하면 된다.

```kotlin
class YamlPropertySourceFactory : PropertySourceFactory {

    override fun createPropertySource(name: String?, encodedResource: EncodedResource): PropertySource<*> {

        encodedResource.resource.filename?.let {
            val factory = YamlPropertiesFactoryBean()
            factory.setResources(encodedResource.resource)
            factory.afterPropertiesSet()

            return PropertiesPropertySource(it, factory.getObject()!!)
        }

        throw IllegalArgumentException("Resource file name is null.")
    }
}
```

## 테스트에 적용

앞서 살펴본 것처럼 `@PropertySource`로 yml 파일 위치와 팩토리 클래스를 지정하면 된다

```kotlin
@PropertySource("classpath:/config/application.yml", factory = YamlPropertySourceFactory::class)
@SpringBootTest  // 또는 @WebMvcTest, @DataJpaTest, ...
class XXX {
    ...
}
```

참고로 `@TestPropertySource`로는 실제 yml 파일 위치를 `locations` 속성으로 지정하더라도 파일을 찾지 못한다.

