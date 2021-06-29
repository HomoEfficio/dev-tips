# OpenAPI - Spring Pageable

스프링을 사용해서 페이징 구현 시 클라이언트로부터의 페이징 정보를 보통 다음과 같이 전달받는다.

```kotlin
fun getPagedResult(pageable: Pageable): ResponseEntity<Page<XXX>>
```

이걸 그대로 springdoc 을 사용해서 openapi를 만들어내면 다음과 같이 나온다.

```yml
...
      parameters:
      - name: pageable
        in: query
        required: true
        schema:
          $ref: '#/components/schemas/Pageable'
...

components:
  schemas:
    Pageable:
      type: object
      properties:
        pageNumber:
          type: integer
          format: int32
        pageSize:
          type: integer
          format: int32
        sort:
          $ref: '#/components/schemas/Sort'
        offset:
          type: integer
          format: int64
        paged:
          type: boolean
        unpaged:
          type: boolean
    Sort:
      type: object
      properties:
        sorted:
          type: boolean
        unsorted:
          type: boolean
        empty:
          type: boolean
```

그리고 swagger ui 에도 다음과 같이 나오며,

![Imgur](https://i.imgur.com/pbrifAN.png)


이대로 실행해보면 아래와 같이 불필요한 sort 관련 문자열이 포함돼서 요청이 전송되고,

```
http://localhost:8082/api/.......&sort[sorted]=true&sort[unsorted]=true&sort[empty]=true&offset=0&pageNumber=0&pageSize=10&paged=true&unpaged=true
```

서버에서는 아래와 같은 에러가 발생한다.

```
2021-06-29 18:14:48.107 DEBUG 67129 --- [io-8082-exec-10] o.apache.coyote.http11.Http11Processor   : Error parsing HTTP request header

java.lang.IllegalArgumentException: Invalid character found in the request target [/api/.......&sort[sorted]=true&sort[unsorted]=true&sor.......
```

swagger ui 에도 다음과 같이 보기 흉한 응답이 표시된다.

![Imgur](https://i.imgur.com/BlXffML.png)


일반적으로 사용하는 것처럼 page, size, sort 정도만 parameter 정보로 제공되게 하려면 어떻게 해야할까?

어차피 springdoc 도구를 사용해서 자동 생성되는 openapi 문서니까 해법도 springdoc 에서 찾아야 할 것이다. 고맙게도 [2019년](https://github.com/springdoc/springdoc-openapi/issues/177)부터 답이 나와 있었다. 


## springdoc-openapi-data-rest

`springdoc-openapi-data-rest` 의존 관계를 추가하고,

```kotlin
dependencies {
...
    implementation("org.springdoc:springdoc-openapi-ui:1.5.9")
    implementation("org.springdoc:springdoc-openapi-data-rest:1.5.9")  // 여기!!
...

```

소스 코드에서 `@ParameterObject`를 Pageable 변수 앞에 붙여주면 된다.

```kotlin
fun getPagedResult(@ParameterObject pageable: Pageable): ResponseEntity<Page<XXX>>
```

이제 openapi 문서를 생성해보면 다음과 같이 `Pageable` 객체에 대한 내용 대신 page, size, sort 내용이 표시된다.

```yml
...
      parameters:
      - name: page
        in: query
        description: Zero-based page index (0..N)
        required: false
        schema:
          minimum: 0
          type: integer
          default: 0
      - name: size
        in: query
        description: The size of the page to be returned
        required: false
        schema:
          minimum: 1
          type: integer
          default: 20
      - name: sort
        in: query
        description: "Sorting criteria in the format: property(,asc|desc). Default\
          \ sort order is ascending. Multiple sort criteria are supported."
        required: false
        schema:
          type: array
          items:
            type: string
...
```

swagger ui 에서도 의도했던대로 page, size, sort 만 표시되고,

![Imgur](https://i.imgur.com/VghwNQN.png)


실행하면 의도한대로 페이징된 결과가 나온다.

![Imgur](https://i.imgur.com/3kSfeJF.png)

