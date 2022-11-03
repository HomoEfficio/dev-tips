# RestTemplate으로 대용량 데이터 전송

일반적인 RestTemplate을 사용하면 OOM이 발생하므로 다음과 같이 RestTemplate을 별도로 생성해서 사용하면 된다.

```kotlin
    private fun getRestTemplateForLargeFileTransfer(): RestTemplate {
        val simpleClientHttpRequestFactory = SimpleClientHttpRequestFactory()
        simpleClientHttpRequestFactory.setBufferRequestBody(false)
        return RestTemplate(simpleClientHttpRequestFactory)
    }
```

이유는 나중에 적기로 ㅋ

