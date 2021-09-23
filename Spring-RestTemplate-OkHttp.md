# Spring RestTemplate OkHttp

스프링 WebMVC 에서 외부 서버를 호출할 때 RestTemplate을 많이 사용한다.

RestTemplate 은 내부적으로 java.net.HttpUrlConnection 을 사용하는데, HTTP PATCH method 를 사용하면 다음과 같은 예외가 발생한다.

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

HttpUrlConnection의 소스를 잠시 살펴보면 아래와 같이 PATCH 가 없다는 걸 확인할 수 있다.

```java
    ...
    /* valid HTTP methods */
    private static final String[] methods = {
        "GET", "POST", "HEAD", "OPTIONS", "PUT", "DELETE", "TRACE"
    };
    ...
```

RestTemplate을 사용하면서 PATCH 메서드를 사용하려면 어떻게 해야할까? 여러 방법이 있겠지만, 널리 사용되는 OkHttp 를 사용하는 방법을 기록해둔다.

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


