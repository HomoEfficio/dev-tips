# Spring RestTemplate PATCH Method

아래와 같은 에러가 발생했다.

```
org.springframework.web.client.ResourceAccessException: I/O error on PATCH request for "http://localhost:8081/ㅋㅋㅋㅋㅋ": Read timed out; nested exception is java.net.SocketTimeoutException: Read timed out
```

분명히 PATCH 메서드로 열려 있는 api 라능..

```
2021-07-21 17:51:47.209 DEBUG 70085 --- [  restartedMain] _.s.web.servlet.HandlerMapping.Mappings  : 
    m.z.s.w.c.c.c.InternalController:
    {PATCH [/ㅋㅋㅋㅋㅋ], produces [application/json]}: reviewStart(long,long)
```

뭔가 하고 찾다 보니 이런 어처구니 없는.. 머선 일이고..

![Imgur](https://i.imgur.com/BT0lnrd.png)

링크: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html#patchForObject-java.lang.String-java.lang.Object-java.lang.Class-java.util.Map-

문서에 `You need to use the Apache HttpComponents or OkHttp request factory.`라고 써있는데 뭘 잘못했는지 Apache HttpComponents 는 마찬가지 문제가 발생해서 OkHttp 를 사용해서 다음과 같이 조치하면, PATCH 호출도 잘 된다.

```kotlin
// build.gradle.kts
implementation("com.squareup.okhttp3:okhttp:4.9.1")

// RestTemplate Bean 생성
    @Bean
    fun restTemplate(builder: RestTemplateBuilder): RestTemplate {
        val httpClient = OkHttpClient.Builder()
            .connectTimeout(Duration.ofMillis(connectionTimeout))
            .readTimeout(Duration.ofMillis(readTimeout))
            .build()

        return builder
            .requestFactory { OkHttp3ClientHttpRequestFactory(httpClient) }
            .build()
    }
```


