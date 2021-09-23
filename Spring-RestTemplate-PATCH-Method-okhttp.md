# Spring RestTemplate PATCH Method OkHttp

스프링 WebMVC 에서 외부 서버를 호출할 때 RestTemplate을 많이 사용한다. 그런데 아래와 같은 에러가 발생했다.

```
org.springframework.web.client.ResourceAccessException: I/O error on PATCH request for "http://localhost:8081/ㅋㅋㅋㅋㅋ": Read timed out; nested exception is java.net.SocketTimeoutException: Read timed out
```
또는 이런 에러가 나기도 한다.

```
Caused by: java.net.ProtocolException: Invalid HTTP method: PATCH
	at java.base/java.net.HttpURLConnection.setRequestMethod(HttpURLConnection.java:487) ~[na:na]
	at java.base/sun.net.www.protocol.http.HttpURLConnection.setRequestMethod(HttpURLConnection.java:570) ~[na:na]
	at java.base/sun.net.www.protocol.https.HttpsURLConnectionImpl.setRequestMethod(HttpsURLConnectionImpl.java:370) ~[na:na]
	at org.springframework.http.client.SimpleClientHttpRequestFactory.prepareConnection(SimpleClientHttpRequestFactory.java:217) ~[spring-web-5.3.9.jar:5.3.9]
	at org.springframework.http.client.SimpleClientHttpRequestFactory.createRequest(SimpleClientHttpRequestFactory.java:146) ~[spring-web-5.3.9.jar:5.3.9]
	at org.springframework.http.client.support.HttpAccessor.createRequest(HttpAccessor.java:124) ~[spring-web-5.3.9.jar:5.3.9]
	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:772) ~[spring-web-5.3.9.jar:5.3.9]
	... 14 common frames omitted
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

RestTemplate 은 내부적으로 java.net.HttpUrlConnection 을 사용하는데, HTTP PATCH method 를 지원하지 않는다 --;

HttpUrlConnection의 소스를 잠시 살펴보면 아래와 같이 PATCH 가 없다는 걸 확인할 수 있다.

```java
    ...
    /* valid HTTP methods */
    private static final String[] methods = {
        "GET", "POST", "HEAD", "OPTIONS", "PUT", "DELETE", "TRACE"
    };
    ...
```


그래서 문서에 `You need to use the Apache HttpComponents or OkHttp request factory.`라고 써있기도 하다. 뭘 잘못했는지 Apache HttpComponents 는 마찬가지 문제가 발생해서 OkHttp 를 사용해서 다음과 같이 조치하면, RestTemplate을 사용해서 PATCH 호출도 잘 된다.

### OkHttp 의존관계 추가

```kotlin
// build.gradle.kts
    ...
    implementation("com.squareup.okhttp3:okhttp:4.9.1")
    implementation("com.squareup.okhttp3:logging-interceptor:4.9.1")
    ...
```

### RestTemplate 에 OkHttp 주입

```kotlin
@Configuration
class WebClientConfiguration(
    private val env: Environment,
    @Value("\${http.timeout.connection}")
    private val connectionTimeout: Long,
    @Value("\${http.timeout.read}")
    private val readTimeout: Long,
) {

    @Bean
    fun okHttpRestTemplate(builder: RestTemplateBuilder): RestTemplate {
        // 기본 httpclient 는 HttpMethod.PATCH 를 사용하지 못함 -> okhttp3 로 변경
        val httpLoggingInterceptor = HttpLoggingInterceptor()
        httpLoggingInterceptor.setLevel(okHttpLoggingLevel())

        log.info("===== okhttp log level: [${httpLoggingInterceptor.level}]")

        val httpClient = OkHttpClient.Builder()
            .connectTimeout(Duration.ofMillis(connectionTimeout))
            .readTimeout(Duration.ofMillis(readTimeout))
            .protocols(listOf(Protocol.HTTP_1_1))
            .addInterceptor(httpLoggingInterceptor)
            .build()

        return builder
            .requestFactory { OkHttp3ClientHttpRequestFactory(httpClient) }
            .build()
    }

    private fun okHttpLoggingLevel(): HttpLoggingInterceptor.Level {
        return if (env.acceptsProfiles(Profiles.of("prod"))) HttpLoggingInterceptor.Level.HEADERS
        else HttpLoggingInterceptor.Level.BODY
    }

    private val log = LoggerFactory.getLogger(javaClass)
}

```
