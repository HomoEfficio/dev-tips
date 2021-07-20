# RestTemplate Generic ClassCastException

스프링에서 RestTemplate 을 사용하면 overload 된 여러 방식이 있지만, 대략 아래와 같이 응답 본문(response body)의 타입을 지정해서 받을 수 있다.

```kotlin
responseEntity = restTemplate.exchange(
    uri, httpMethod,
    httpEntity,
    object: ParameterizedTypeReference<MyResponseBodyType>() {}
)
```

## 문제

그런데 실무에서는 restTemplate 을 사용해서 외부 api 를 호출하는 부분은 공통 로직화해서 사용하는 경우가 많다.  
여러 방법이 있겠지만 대략 다음과 같은 공통 로직을 만들 수 있다. 이 때 응답 본문 타입을 특정할 수 없으므로 제네릭을 사용하게 된다.

```kotlin
fun <T> invokeRemoteApi(
    restTemplate: RestTemplate,
    httpMethod: HttpMethod,
    url: String,
    authKey: String,
    reqBody: Any?,
): ResponseEntity<T> {  // 여기!! 제네릭 사용
    val httpHeaders = HttpHeaders()
    httpHeaders.add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    httpHeaders.add(HttpHeaders.AUTHORIZATION, "Bearer $authKey")
    val uri = URI.create(url)
    val httpEntity = HttpEntity(reqBody, httpHeaders)
    log.debug("invoking remote api, method: [$httpMethod], url: [$url], authKey: [$authKey], reqBody: [$reqBody]")
    var responseEntity: ResponseEntity<T>? = null  // 여기!! 제네릭 사용
    try {
        responseEntity = restTemplate.exchange(
            uri, httpMethod,
            httpEntity,
            object: ParameterizedTypeReference<T>() {}  // 여기!! 제네릭 사용
        )
    } catch (t: Throwable) {
        log.error("invoking remote api error!!", t.fillInStackTrace())
    }
    if (!responseEntity!!.statusCode.is2xxSuccessful) {
        throw RemoteApiInvocationFailureException(
            uri,
            httpMethod,
            httpEntity,
        )
    }
    return responseEntity
}
```

컴파일에는 아무런 문제가 없지만 이렇게 하면 실제 실행 시 다음과 같이 ClassCastException 이 발생한다.

```
java.lang.ClassCastException: java.util.LinkedHashMap cannot be cast to MY_RESPONE_BODY_CLASS
```

이유는 정확하진 않지만 아마도 타입 소거 때문인 것 같다.

이럴 때는 어떻게 해야할까?


## 해결

ParameterizedTypeReference 에는 `forType(Type)` 이라는 static method 가 있다. 이 메서드를 사용하면 타입을 명시적으로 전달해줄 수 있을 것 같다.  
그래서 아래와 같이 변경하고 테스트해보니 ClassCastException 이 발생하지 않고 MyResponseBodyType 타입으로 잘 받아온다.

```kotlin
fun <T> invokeRemoteApi(
        restTemplate: RestTemplate,
        httpMethod: HttpMethod,
        url: String,
        authKey: String,
        reqBody: Any?,
        type: Type? = null,  // 여기!! 타입을 명시적으로 전달 받는다. 응답 본문이 필요하지 않아 타입 지정이 필요 없다면 null 전달
    ): ResponseEntity<T> {
        val httpHeaders = HttpHeaders()
        httpHeaders.add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        httpHeaders.add(HttpHeaders.AUTHORIZATION, "Bearer $authKey")
        val uri = URI.create(url)
        val httpEntity = HttpEntity(reqBody, httpHeaders)
        log.debug("invoking remote api, method: [$httpMethod], url: [$url], authKey: [$authKey], reqBody: [$reqBody]")
        var responseEntity: ResponseEntity<T>? = null
        try {
            responseEntity = restTemplate.exchange(
                uri, httpMethod,
                httpEntity,
                if (type != null) ParameterizedTypeReference.forType(type)  // 여기!! forType 을 사용해서 명시적으로 타입을 전달
                    else object: ParameterizedTypeReference<T>() {}         // 여기!! 응답 본문이 필요하지 않을 때는 제네릭으로 그냥 컴파일 에러만 회피
            )
        } catch (t: Throwable) {
            log.error("invoking remote api error!!", t.fillInStackTrace())
        }
        if (!responseEntity!!.statusCode.is2xxSuccessful) {
            throw RemoteApiInvocationFailureException(
                uri,
                httpMethod,
                httpEntity,
            )
        }
        return responseEntity
    }
```

위 함수를 호출하는 쪽 코드는 아래와 같다.

```kotlin
// 응답 본문 타입 지정
val responseEntity: ResponseEntity<YYY> = invokeRemoteApi(
    restTemplate = restTemplate,
    httpMethod = HttpMethod.POST,
    url = REMOTE_API_URL,
    authKey = AUTH_KEY,
    reqBody = XXX,
    type = YYY::class.java,
)

// 응답 본문 타입 미지정
val response = invokeRemoteApi<Nothing>(
    restTemplate = restTemplate,
    httpMethod = HttpMethod.POST,
    url = REMOTE_API_URL,
    authKey = AUTH_KEY,
    reqBody = XXX,
    type = null
)
```
