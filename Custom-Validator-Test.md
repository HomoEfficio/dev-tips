# Custom Validator Unit Test

javax.validation.ConstraintValidator 를 구현하면 쉽게 커스텀 validator 를 만들 수 있다.  
그리고 스프링 애플리케이션 컨텍스트 없이도 가볍게 테스트를 돌릴 수 있다.  

한 가지 조건이 있는데, 라기보다는 있는 걸로 보이는데, validator 객체가 기본 생성자로만 생성이 되는 것 같다.  
생성자에 파라미터를 추가하면 아래 위치에서 NoSuchMethodException 이 발생한다.

![Imgur](https://i.imgur.com/hl7SWyW.png)

```
javax.validation.ValidationException: HV000064: Unable to instantiate ConstraintValidator: XXXValidator.

...
Caused by: java.lang.NoSuchMethodException: XXXValidator.<init>()

```

기본 생성자로만 가능하다면, 인자를 받아서 구현할 필요가 있는 상황에서는 어떻게 해야할까?  
예를 들어 CommunitySift API를 활용해서 호출해서 텍스트 검수를 해야하는 상황이고,  
커스텀 validator를 사용해서 검수한다면 API URL이 필요하다. 기본 생성자 밖에 없는 데 URL 값을 validator에 어떻게 심어줄 수 있을까?


## Annotation Attribute 활용

아래와 같이 애노테이션 애튜리뷰트에 파라미터를 정의하고 ConstraintValidator 의 initialize() 메서드를 재정의하면 된다.

```kotlin
@Target(AnnotationTarget.FIELD, AnnotationTarget.PROPERTY, AnnotationTarget.VALUE_PARAMETER, AnnotationTarget.TYPE)
@Constraint(validatedBy = [XXXValidator::class])
@MustBeDocumented
annotation class XXXFilter(
    val message: String = "xxx_text_filter",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = [],
    val apiUrl: String,  // 필요한 파라미터 정의
    val authToken: String = "xxxyyyzzz",  // 필요한 파라미터 기본값 정의. 기본값이 있더라도 애너테이션 사용 시 지정한 값이 우선
)


class XXXValidator : ConstraintValidator<XXXFilter, Any> {

    private lateinit var apiUrl: String  // 인자값을 받을 변수
    private lateinit var authToken: String  // 인자값을 받을 변수
    private var restTemplate: RestTemplate = restTemplate()  // restTemplate 은 자체 생성
    
    private val log = LoggerFactory.getLogger(javaClass)

    override fun initialize(constraintAnnotation: XXXFilter) {
        apiUrl = constraintAnnotation.apiUrl  // 애노테이션으로부터 인자값 받아서 주입
        authToken = constraintAnnotation.authToken  // 애노테이션으로부터 인자값 받아서 주입
    }

    override fun isValid(value: Any?, context: ConstraintValidatorContext?): Boolean {
        ...
    }


    private fun httpClient(): OkHttpClient {
        val httpLoggingInterceptor = HttpLoggingInterceptor()
        httpLoggingInterceptor.setLevel(HttpLoggingInterceptor.Level.HEADERS)

        return OkHttpClient.Builder()
            .connectTimeout(Duration.ofMillis(3000))
            .readTimeout(Duration.ofMillis(3000))
            .protocols(listOf(Protocol.HTTP_1_1))
            .addInterceptor(httpLoggingInterceptor)
            .build()
    }

    private fun restTemplate(): RestTemplate {
        return RestTemplate(OkHttp3ClientHttpRequestFactory(httpClient()))
    }
}
```

## Annotation 사용

애노테이션은 다음과 같이 사용하면 된다.

```kotlin
data class ZzzIn(
    ...

    @field: [
        NotNull(message = "{err.client.field_required}")
        NotBlank(message = "{err.client.field_required}")
        Size(max = WorldVersion.MAX_DESCRIPTION, message = "{err.client.illegal_size}")
        XXXFilter(message = "alert_text_filter", apiUrl = "https://aaa.bbb.ccc")  // 여기!!
    ]
    val name: String,

    ...
)

```

## Test

앞에서 말한 것처럼 ConstraintValidator 는 javax.validation 패키지에 있으며 스프링에 대한 의존성이 없다.  
그래서 아래와 같이 아무런 의존 관계 구성할 필요 없이 `Validation.buildDefaultValidatorFactory().validator`를 사용해서 가볍게 테스트 할 수 있다.

```kotlin
internal class XXXValidatorTest {

    @Test
    internal fun `name 필드값이 XXXFilter 에 위반하면 ConstraintViolation 갯수는 1이다`() {
        val validator = Validation.buildDefaultValidatorFactory().validator

        val zzzIn = ZzzIn(
            name = "abcde",
            description = "abcde description",
        )

        val result: MutableSet<ConstraintViolation<ZzzIn>> = validator.validate(zzzIn)

        assertThat(result.size).isEqualTo(1)
    }
}
```

## 내부 살짝 들여다보기

커스텀 Validator 가 어떻게 로딩돼서 테스트에 사용되는지 살짝 들여다보자.

아래와 같이 ValidatorImpl 클래스에서 validation 대상 필드에 붙어 있는 애노테이션 정보를 모두 수집하고,

![Imgur](https://i.imgur.com/VE5nZyH.png)

아래와 같이 애노테이션 정의 시 `@Constraint(validatedBy = [XXXValidator::class])` 로 지정한 validator 클래스의 인스턴스를 생성한다.

![Imgur](https://i.imgur.com/wk2XN4l.png)

그 후에는 생성된 validator 인스턴스를 활용해서 validation 을 수행한다.

참고로 동일한 타입의 필드를 validate 할 때는 한 번 생성된 인스턴스가 계속 재사용되지만, 다른 타입의 필드를 validate 할 때는 재사용되지 않고 validator 인스턴스가 새로 생성된다.


## 마무리

>- javax.validation.ConstraintValidator 를 구현해서 쉽게 커스텀 validator 를 만들 수 있다.
>- 커스텀 validator 가 정상 동작하려면 기본 생성자가 있어야 한다.
>- 커스텀 validator 에 어떤 값을 주입하려면 애노테이션 애트리뷰트와 validator 의 initialize() 메서드를 활용한다.
>- 다른 의존 관계 필요 없이 `Validation.buildDefaultValidatorFactory().validator`를 사용해서 커스텀 validator 를 가볍게 테스트할 수 있다.
>- 커스텀 validator 인스턴스는 동일한 타입 필드를 validate 할 때는 재사용되지만, 다른 타입 필드를 validate 할 때는 새 인스턴스가 생성된다.

