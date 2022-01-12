# RestTemplate URL OK Test

테스트에서 RestTemplate으로 URL에 자원이 존재하는지 여부 확인할 때 사용하는 code snippet

```kotlin
    // val httpStatus = getResourceHttpStatus(url)
    // assertThat(httpStatus!!.is2xxSuccessful).isEqualTo(true)
    // 와 같이 사용
    private fun getResourceHttpStatus(url: String): HttpStatus? {
        val requestCallback = RequestCallback { request: ClientHttpRequest ->
            request
                .headers.accept = listOf(MediaType.APPLICATION_OCTET_STREAM, MediaType.ALL)
        }

        val responseExtractor: ResponseExtractor<HttpStatus> = ResponseExtractor { response: ClientHttpResponse ->
            response.statusCode
        }
        
        return testRestTemplate.execute(
            url, HttpMethod.GET,
            requestCallback, responseExtractor
        )
    }    
```
