# OpenAPI UI - Authorize

보안이 적용돼 있는 API 를 OpenAPI UI 에서 호출하려면 어떻게 해야할까?

OpenAPI 버전 3 현재 아래 그림과 같이 마치 로그인 한 것처럼 보안에 필요한 값을 미리 넣어주고 api 를 호출할 수 있는 기능이 있다.

![Imgur](https://i.imgur.com/q7oX4mt.png)

![Imgur](https://i.imgur.com/9EQhUd6.png)

따라서 위 그림과 같이 Authorize 버튼이 나오도록 OpenAPI 문서를 작성하고,  
JWT 토큰처럼 보안에 필요한 정보를 생성해주는 API 를 하나 추가하면,  
OpenAPI UI 화면에서 API를 통해 토큰을 만들고 이를 Authorize 팝업창에 입력해서 이후 API 호출에 사용할 수 있다.

JWT를 사용하는 사례로 한 번 살펴보자.

## Authorize 버튼 생성

OpenAPI 문서에 다음 내용을 추가하면 OpenAPI UI 에 위에서 봤던 그림처럼 Authorize 버튼과 팝업 모달을 사용할 수 있게 된다.

`Bearer` 항목은 임의로 정할 수 있는 이름으로서 보안 정책 이름이라고 생각하면 쉽다. 빨간줄로 표시한 두 곳의 값이 같기만 하면 된다.

![Imgur](https://i.imgur.com/FXWkoZx.png)

또는 OpenAPI yaml 문서 대신 아래와 같이 소스 코드를 통해 설정할 수도 있다.

```kotlin
@Configuration
@OpenAPIDefinition(
    servers = [
        Server(url = "/any-context-root")
    ],
    security = [
        SecurityRequirement(name = "Bearer"),
    ]
)
@SecuritySchemes(
    SecurityScheme(
        type = SecuritySchemeType.HTTP,
        name = "Bearer",
        description = "JWT Bearer Token 입력",
        scheme = "bearer",
        bearerFormat = "JWT"
    ),
)
class OpenApiConfig {
}

```
`security`가 배열로 정의돼 있고, `@SecuritySchemes`도 복수의 `@SecurityScheme`을 가질 수 있으므로,  
예를 들어 사용자 인증과 서버간 인증이 별도로 존재할 때 2개의 인증 방식을 사용할 수도 있다.

복수개로 구성 후 Authorize 버튼을 누르면 다음과 같이 2가지가 함께 표시된다.

![Imgur](https://i.imgur.com/jO3Q7L9.png)


## JWT 생성 API 추가

다음과 같이 JWT 를 만들어내는 API 를 추가한다.

JWT 생성 API 를 운영 환경에서도 공개하면 안 되므로, `@ConditionalOnProperty`를 사용해서 properties 값을 기준으로 활성화 여부를 제어한다.

```kotlin
@ConditionalOnProperty(name = ["jwt.generator.enabled"], havingValue = "true")
@RequestMapping("/jwt-token")
@RestController
class AController(
    private val jwtProcessor: JwtProcessor
) {

    @GetMapping
    fun jwtToken(username: String): ResponseEntity<String> {
        return ResponseEntity.ok(jwtProcessor.createToken(username))
    }
}
```

JWT 토큰을 생성하는 로직은 대략 다음과 같다. 여러 도구가 있지만 Jwts 를 사용했다.

```kotlin
@Component
class JwtProcessor(
    private val jwtProperties: JwtProperties
) {
    val key: SecretKey = Keys.hmacShaKeyFor(Decoders.BASE64.decode(jwtProperties.base64EncodedSecret))

    fun createToken(username: String): String? {
        return Jwts.builder()
            .setSubject("Any Sub Value")
            .claim(JwtKeys.AUTHORITIES_KEY.keyName, "ROLE_USER")
            .claim(JwtKeys.UID.keyName, username)
            .signWith(key, SignatureAlgorithm.HS512)
            .setExpiration(Date(Date().time + (1000 * 60 * 30)))
            .compact()
    }
```

## OpenAPI UI 에서 JWT 토큰 생성

위와 같이 API 를 추가하면 다음과 같이 OpenAPI UI 에 API 가 표시되며, 필요한 입력값을 넣고 JWT 토큰을 얻을 수 있다.

![Imgur](https://i.imgur.com/ctKkENh.png)


## Authorize 팝업 화면에 JWT 입력

Authorize 버튼을 클릭하면 나오는 팝업 화면에서 방금 전에 복사해둔 JWT 값을 입력하고 Authorize 를 클릭한다.

![Imgur](https://i.imgur.com/OU93IXD.png)

Close 를 클릭해서 팝업을 제거한다.

![Imgur](https://i.imgur.com/Bw4EhjI.png)


## 보안 적용 API 호출

이제 보안이 적용된 API 를 호출해보면 다음과 같이 JWT 값이 Authorization 헤더에 Bearer 토큰 값으로 포함되며 보안을 통과해서 API 호출에 성공한다.

![Imgur](https://i.imgur.com/KgjGEWY.png)
