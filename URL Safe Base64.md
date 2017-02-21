# URL Safe Base64

값에 괄호가 들어가 있는 쿠키는 브라우저에서는 잘 보여도, 톰캣에서 `request.getCookies()`로 읽으면 보이지 않는다.

인코딩을 해줘야 할 것 같은데, URL인코딩은 `%`가 들어갈테니 또 특수 문자라고 안 받아줄 것 같고, Base64를 쓰자니 `+`와 `/`가 걸리고..

찾아보니 URLSafeBase64라는게 있었다. 정확하게는 [Base 64 Encoding with URL and Filename Safe Alphabet](https://tools.ietf.org/html/rfc4648#page-7)인데, Base64 인코딩의 단점인 `+`와 `/`로 인코딩 되는 것을 `-`와 `_`로 인코딩 되게 해준다.

## node.js의 urlsafe-base64

https://www.npmjs.com/package/urlsafe-base64에 node.js 용 URLSafeBase64 모듈이 있다.

사용법은 간단하지만, 한 가지 신경 써줄 것은 인코딩 할 문자열을 Buffer로 감싸줘야 한다는 것이다.

```javascript
var urlsafe_base64 = require('urlsafe-base64');
var DMP_UID_VALUE = urlsafe_base64.encode(new Buffer(req.param(DMP_UID_NAME)));
```

## Java8의 Base64

Java8 이전에는 [Apache commons의 Base64](https://commons.apache.org/proper/commons-codec/apidocs/org/apache/commons/codec/binary/Base64.html)를 써야 URLSafeBase64 인코딩이 가능했는데, Java8에 포함된 `java.util.Base64`는 자체적으로 URLSafeBase64를 지원한다.

그래서 아래와 같이 인코딩 해서 node.js의 `urlsafe-base64`와 비교 테스트 할 수 있고,

```java
import java.util.Base64;
import static java.nio.charset.StandardCharsets.UTF_8;

String rawCookieValue = "(DMPD)b80f9ed8-4e66-41a4-ac60-b46ea5586cf0";
byte[] urlSafeBase64Encoded = Base64.getUrlEncoder().encode(rawCookieValue.getBytes(UTF_8));
String encodedCookieValue = new String(urlSafeBase64Encoded, UTF_8);
System.out.println(encodedCookieValue);

String encodedFromNodeJs = "KERNUEQpYjgwZjllZDgtNGU2Ni00MWE0LWFjNjAtYjQ2ZWE1NTg2Y2Yw";
System.out.println(encodedFromNodeJs);
System.out.println("nodejs의 urlsafe-base64 인코딩값 == java8의 urlsafe-base64 인코딩값 : " +
        encodedCookieValue.equals(encodedFromNodeJs));
```

아래와 같이 디코딩 하고 테스트 할 수 있다.

```java
// 위에 있는 인코딩 소스에 이어서..

byte[] decoded = Base64.getUrlDecoder().decode(encodedCookieValue.getBytes(UTF_8));
String urlSafeBase64Decoded = new String(decoded, UTF_8);

System.out.println("++++++++++++");
System.out.println(rawCookieValue);
System.out.println(urlSafeBase64Decoded);
System.out.println("인코딩 전 쿠키값 == 디코딩 후 쿠키값 : " + rawCookieValue.equals(urlSafeBase64Decoded));
```

특이한 점은 그냥 사용할 때는 try-catch를 해주지 않아도 되는데, JUnit 테스트에서 멀쩡하던 테스트들이 위 코드로 인해 오류가 나는 현상이 있다.

이유는 확실히 모르지만 일단 위의 인코딩/디코딩 부분을 `try { ... } catch (Exception e) {}`로 묶어주면 테스트 중 에러가 발생하지 않는다.
