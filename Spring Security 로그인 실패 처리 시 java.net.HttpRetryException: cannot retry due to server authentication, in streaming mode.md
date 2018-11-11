# Spring Security 로그인 실패 처리 시 java.net.HttpRetryException: cannot retry due to server authentication, in streaming mode

## 상황

잘못된 username이나 password로 로그인 시도 시 스프링 시큐리티 내부적으로 `BadCredentialsException`을 생성해서 예외를 던진다.

`BadCredentialsException`을 별도로 처리하지 않으면 클라이언트에게는 `401 UNAUTHORIZED`가 아니라 `400 BAD_REQUEST`가 반환된다.

그래서 `401 UNAUTHORIZED`을 반환하기 위해 예를 들어 다음과 같이 `@ControllerAdvice`가 지정된 클래스에서 처리해주면,

```java
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    @ExceptionHandler(BadCredentialsException.class)
    public ApiError handleBadCredentialsException(BadCredentialsException e) {
        return new ApiError(HttpStatus.UNAUTHORIZED, LocalDateTime.now().toString(), e.getMessage());
    }
```

이번에는 다음과 같이 영 뜬금없는 예외가 발생한다.

```
org.springframework.web.client.ResourceAccessException: I/O error on POST request for "http://localhost:51180/어쩌구/엔드포인트": cannot retry due to server authentication, in streaming mode; nested exception is java.net.HttpRetryException: cannot retry due to server authentication, in streaming mode

	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:743)
	at org.springframework.web.client.RestTemplate.execute(RestTemplate.java:669)
	at org.springframework.web.client.RestTemplate.postForEntity(RestTemplate.java:444)
	at org.springframework.boot.test.web.client.TestRestTemplate.postForEntity(TestRestTemplate.java:502)

  .. 중략 ..
  
Caused by: java.net.HttpRetryException: cannot retry due to server authentication, in streaming mode
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1638)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1441)
	at java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:480)
	at org.springframework.http.client.SimpleClientHttpResponse.getRawStatusCode(SimpleClientHttpResponse.java:55)
	at org.springframework.web.client.DefaultResponseErrorHandler.hasError(DefaultResponseErrorHandler.java:51)
	at org.springframework.web.client.RestTemplate.handleResponse(RestTemplate.java:765)
	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:735)
	... 36 more
```

## 원인

검색을 좀 해보니 이미 2015년에 [관련 이슈](https://github.com/spring-projects/spring-security-oauth/issues/441#issuecomment-92033542)가 있었던 모양이다.

대충 보니, 담당인 Dave Syer 말로는 스프링에서 사용하는 `SimpleHttpClientRequestFactory`가 JDK의 `com.sun.* HTTP`에 의존하는데, 그 HTTP Client가 이미 실패한 요청의 Body에 접근할 수 없게 되어 있어서 위와 같은 에러가 발생한다고 한다. 그리고 스펙(무슨 스펙인지는 정확히 모르겠..)대로라면 인증 실패 시 400으로 반환하게 되어 있어서 401이 아니라 400을 반환하게 구현한 모양이다.

## 처리

두 가지가 있다.

1. BadCredentialsException에 대해 별도의 처리를 하지 않고 내버려 두기

    별도의 처리를 하지 않으면 Body를 다시 읽을 일이 없으므로 위와 같은 `HttpRetryException`는 발생하지 않고, 다음과 같은 응답이 클라이언트에게 반환된다.

    ```
    <400,{status=BAD_REQUEST, timestamp=2018-11-11T16:48:50.053, message=Bad credentials, debugMessage=null, fieldErrors=null},{X-Content-Type-Options=[nosniff], X-XSS-Protection=[1; mode=block], Cache-Control=[no-cache, no-store, max-age=0, must-revalidate], Pragma=[no-cache], Expires=[0], X-Frame-Options=[DENY], Content-Type=[application/json;charset=UTF-8], Transfer-Encoding=[chunked], Date=[Sun, 11 Nov 2018 07:48:49 GMT], Connection=[close]}>
    ```

1. BadCredentialsException에 대해 별도의 처리를 하고, Apache HttpClient를 사용한다.

    Apache HttpClient를 사용하면 다음과 같이 원하는대로 401 이 반환되게 할 수 있다.
    
    ```
    <401,{status=UNAUTHORIZED, timestamp=2018-11-11T16:55:31.148, message=Bad credentials, debugMessage=null, fieldErrors=null},{X-Content-Type-Options=[nosniff], X-XSS-Protection=[1; mode=block], Cache-Control=[no-cache, no-store, max-age=0, must-revalidate], Pragma=[no-cache], Expires=[0], X-Frame-Options=[DENY], Content-Type=[application/json;charset=UTF-8], Transfer-Encoding=[chunked], Date=[Sun, 11 Nov 2018 07:55:30 GMT]}>
    ```
