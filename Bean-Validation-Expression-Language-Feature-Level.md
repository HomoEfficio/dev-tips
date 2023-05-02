# Bean Validation Expression Language Feature Level

https://www.baeldung.com/spring-validation-message-interpolation 에 보면 밸리데이션 메시지에 여러 값들을 주입해서 표시할 수 있다.

예를 들어 아래와 같이 밸리데이션 애노테이션을 지정하고,

```kotlin
@field: [
    NotNull(message = "{err.client.field_required}")
    NotBlank(message = "{err.client.field_required}")
    Size(max = WorldVersion.MAX_NAME, message = "{err.client.illegal_size}")
]
val name: String,
```

ValidationMessages_ko_KR.properties 파일에 아래와 Expression Language 를 사용한 메시지를 지정해두면,

```
err.client.illegal_size=길이는 {min} 이상 {max} 이하여야 합니다. 입력값 길이: ${validatedValue.length()}
```

밸리데이션 실패시 아래은 메시지를 표시할 수 있다.

```
길이는 0 이상 100 이하여야 합니다. 입력값 길이: 101
```


## 문제

이게 스프링 부트 2.4 까지는 기본적인 메시지 설정만으로 동작했는데, 스프링 부트 2.5 에서는 아래와 같은 에러가 발생한다.

```
2021-07-31 18:15:07.495 ERROR 58182 --- [           main] o.h.v.i.e.m.ElTermResolver               : HV000264: Unable to interpolate EL expression '${validatedValue.length()}' as it uses a disabled feature.

org.hibernate.validator.internal.engine.messageinterpolation.el.DisabledFeatureELException: Method execution is not supported when only enabling Expression Language bean property resolution.
    at org.hibernate.validator.internal.engine.messageinterpolation.el.BeanPropertiesELResolver.invoke(BeanPropertiesELResolver.java:23) ~[hibernate-validator-6.2.0.Final.jar:6.2.0.Final]
    at javax.el.CompositeELResolver.invoke(CompositeELResolver.java:79) ~[tomcat-embed-el-9.0.46.jar:3.0.FR]
```


## 원인

`Method execution is not supported when only enabling Expression` 메시지가 영 구려서 뭔 뜻인지 금방 알기 어려웠는데,  
알고 보니 메시지에서 `${validatedValue.length()}` 이렇게 `length()` 메서드 호출을 사용했는데, 기본 설정에서는 메서드 호출을 허용하지 않는다는 것이다.  

찾아보니 Expression Language 에도 기능 지원 수준이 있었다. org.hibernate.validator.messageinterpolation.ExpressionLanguageFeatureLevel 에 정의돼 있으며,  
FeatureLevel 기본값은 BEAN_PROPERTIES 이고 이 수준에서는 메서드 호출을 사용할 수 없다.

메서드 호출을 사용하려면 BEAN_METHODS 로 설정해야 하는데, 주석에 보면 메서드 호출이 보안 약점으로 작용할 수도 있다고 한다.  
그러니 웬만하면 메서드 호출을 허용하지 않는 게 좋겠고, 그래도 꼭 해야한다면 FeatureLevel 을 BEAN_METHODS 로 설정해줘야 한다.  

어떻게?


## 설정

아직 @Incubation 이라 그런지 자료가 별로 없는데 메시지 설정 시 아래와 같이 추가해주면 된다.

```kotlin
import org.hibernate.validator.BaseHibernateValidatorConfiguration.CONSTRAINT_EXPRESSION_LANGUAGE_FEATURE_LEVEL
import org.hibernate.validator.messageinterpolation.ExpressionLanguageFeatureLevel
import org.springframework.context.MessageSource
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.support.ReloadableResourceBundleMessageSource
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean

@Configuration
class MessageConfig {
    @Bean
    fun messageSource(): MessageSource {
        val messageSource = ReloadableResourceBundleMessageSource()
        messageSource.setBasename("classpath:message/messages")
        messageSource.setDefaultEncoding("UTF-8")
        return messageSource
    }

    @Bean
    fun getValidator(): LocalValidatorFactoryBean {
        val bean = LocalValidatorFactoryBean()
        bean.setValidationMessageSource(validationMessageSource())
        // 여기!!
        bean.validationPropertyMap[CONSTRAINT_EXPRESSION_LANGUAGE_FEATURE_LEVEL] = ExpressionLanguageFeatureLevel.BEAN_METHODS.name
        return bean
    }

    private fun validationMessageSource(): MessageSource {
        val messageSource = ReloadableResourceBundleMessageSource()
        messageSource.setBasename("classpath:message/validation/ValidationMessages")
        messageSource.setDefaultEncoding("UTF-8")
        return messageSource
    }
}


```

