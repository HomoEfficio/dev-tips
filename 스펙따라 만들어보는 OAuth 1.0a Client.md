# 스펙따라 만들어보는 OAuth 1.0a Client

## OAuth 탄생 배경

- 많은 서비스가 API를 통해 서로 연결

> **내가 A라는 앱에 쓴 글이 내 트위터 타임라인에도 표시되면 좋겠다.**

- 하지만 A 앱은 내 트위터 타임라인에 글을 쓸 권한이 없다.

- 물론 A 앱이 내 트위터 계정 정보를 알고 있다면 A 앱이 내 트위터 타임라인에 글을 쓸 수 있겠지만,

- 필요한 것은 글을 쓸 수 있는 권한 뿐인데 계정 정보를 모두 A 앱에게 알려줄 필요는 없다.

- 따라서 내가 A 앱에게 글을 쓸 권한을 줬다는 사실을 트위터에게 알려주고,

- 그 사실을 트위터가 확인할 수 있다면,

- 내 계정 정보를 A 앱에게 알려주지 않고도 A 앱이 트위터에 글을 쓸 수 있게 된다.

- **내가 A 앱에 쓴 글을 내 트위터 타임라인에도 표시**하려면 결국 다음의 질문에 대한 답이 필요하다.

> **내가 A 앱에게 권한을 줬다는 사실을 트위터에게 어떻게 알려줄 수 있을까?**


# OAuth 1.0a

어떤 행위(내가 A 앱에게 트위터에 글을 쓸 권한을 준 행위)가 이루어졌음을 프로그래밍을 통해 증명하는 여러 방식 중에 대표적으로 서명(Signature)이라는 것이 있다.

아래 그림은 서명 방식 중에서 HMAC(Hashed Message Authentication Code)를 보여주고 있다.

![](https://www.thinqloud.com/wp-content/uploads/2017/07/blog_banner_2-1.jpg)
(출처: https://www.thinqloud.com/hmac-authentication-in-salesforce/)

- 송신자와 수신자가 비밀키를 공유하고,
- 송신자는 평문(빨간 문서 아이콘)과 비밀키를 함께 해시한 값(MAC)과 평문을 함께 수신자에게 보내면,
- 수신자는 송신자가 보낸 평문과 송신자와 공유하고 있는 비밀키를 함께 해시한 값(Hash Output)을 계산하고,
- 계산한 값이 송신자가 보낸 MAC 값과 같은지 비교해서 평문(어떤 행위)이 송신자로부터 전송되었음을 확인한다.

OAuth 1.0a는 권한을 줬다는 사실을 위와 같은 서명 방식을 이용해서 증명한다. 

과정이 조금 복잡한 면이 있어서 요즘은 조금 더 간단하고 편리한 OAuth 2.0이 더 많이 사용되지만, 편리한 만큼 보안성을 양보해야 한다. 

OAuth 1.0a를 이해하면 OAuth 2.0을 쉽게 이해할 수 있으므로 학습 관점에서는 OAuth 1.0a를 먼저 공부하는 것이 의미가 있다.

이제 OAuth 1.0a를 좀더 구체적으로 알아보자.

## 등장 인물(이라 쓰고 용어라고 읽..)

- Resource Owner: 트위터 계정을 가지고 있는 트위터 사용자. 앱 A에 대한 사용권한도 가지고 있다.
- Client: 트위터 API를 이용해서 트위터에 글을 남기려는 앱 A.
- Server: API로 서비스를 제공하는 트위터.

- Client Credentials: 등록을 요청한 Client 앱에게 Server가 발급한 등록 정보
- Temporary Credentials: Client의 권한 부여 요청을 확인하고 Server가 발급한 임시 확인 정보
- Token Credentials: 사용자로부터 권한을 부여받았음을 확인하고 Server가 발급한 접근 토큰 정보

참고로 다음과 같이 가리키는 대상은 같지만 [OAuth 1.0](https://oauth.net/core/1.0/)과 [OAuth 1.0a](https://tools.ietf.org/html/rfc5849)에서의 용어가 다르며, 커뮤니티 버전인 OAuth 1.0의 용어를 그대로 쓰고 있는 자료도 많다.

OAuth 1.0 | OAuth 1.0a
---|---
User | Resource Owner
Consumer | Client
Service Provider | Server
Consumer Key and Secret | Client Credentials
Request Token and Secret | Temporary Credentials
Access Token and Secret | Token Credentials

1.0의 용어가 더 구별하기 쉽고 직관적이어서 학습하기에 좋으므로 본 글에서는 1.0 용어를 사용한다. 1.0 용어로 이해한 후에는 1.0a의 용어도 쉽게 받아들일 수 있을 것이다.

한 가지 짚고 넘어갈 용어로 Secret이 있는데, 서명 방식에서 **Secret은 Service Provider가 Consumer에게 발급하고 둘이 각자 보유하고 있다가 필요할 떄 사용할 뿐 온라인으로 주고 받지 않는 정보**다.

## 사전 조건

- Consumer는 Service Provider의 API를 이용할 수 있도록 등록되어 있어야 한다.
- User는 Consumer와 Service Provider 모두를 사용할 수 있는 권한을 가지고 있다.

## Service Provider로부터 확인 받아야 하는 사항

- Consumer는 User로부터 권한 부여 요청을 받았다는 사실을 Service Provider로부터 확인 받아야 함 - (1)
- User는 Service Provider의 사용자임을 Service Provider로부터 확인 받아야 함 - (2)
- User는 Consumer에게 권한을 부여했음을 Service Provider로부터 확인 받아야 함 - (3)

이 3가지 확인을 받기위한 절차를 개략적으로 생각해보자

## 절차 개요

1. User가 Consumer에 글을 쓰고 'Service Provider에도 남기기' 버튼을 누른다.

1. Consumer는 자신의 등록 정보를 바탕으로 Signature를 만들고 Service Provider에게 Signature를 보내서 사용자로부터 권한 부여 요청을 받았음을 Service Provider에게 알리고, Service Provider는 권한 부여 요청을 확인했다는 임시 증표(Request Token)를 저장하고 Request Token을 Consumer에게 발급한다. (1)

1. Consumer는 권한 부여 요청 확인 증표(Request Token)와 함께 User의 요청을 Service Provider의 인가(권한 부여) 화면으로 리다이렉트한다.

1. User가 Service Provider에 로그인 한 상태가 아니라면 로그인 한다. (2)

1. 인가 화면에는 'Consumer에게 권한 부여' 버튼이 표시된다.

1. User가 'Consumer에게 권한 부여' 버튼을 클릭하면, Service Provider는 User가 (Request Token을 확인하고) Consumer에게 권한을 부여했음을 확인하고, 확인 코드(Verifier 또는 Authorization_code)를 저장 및 User에게 반환하고 Consumer가 제공하는 callback 화면으로 리다이렉트한다. (3)

1. 리다이렉트를 통해 권한 부여 확인 코드를 전달 받은 Consumer는 Consumer Key, Request Token, Verifier 등을 대상으로 Consumer Secret, Request Token Secret를 이용해서 Signature를 만들고 Service Provider에게 Signature를 보낸다.

1. Service Provider는 Consumer가 보낸 Signature를 확인하고 User만 접근할 수 있었던 보호 자원에 대한 접근 증표(Access Token)를 Consumer에게 발급한다.

1. 이후 Consumer는 Access Token를 Service Provider에게 보여주면서 User를 대신해서 보호 자원에 접근한다.

6번까지 진행되면 확인해야 할 3가지 사항은 모두 확인했으므로 바로 보호 자원에 대한 Access Token을 발급할 수 있지만, 6번에서 발급하면 증표가 User에게 직접 발급되고 User의 Local Storage나 Session에 남을 수 있으므로 유출 가능성이 발생한다. 따라서 6번에서는 발급하지 않고 8번에서 Consumer에게 발급한다.

위 과정에서 '권한 부여 요청 확인 증표'를 `Request Token`, '권한 부여 확인 코드'를 `Verifier 또는 AuthorizationCode`, '보호 자원 접근 증표'를 `Access Token`이라고 부른다.

## Sequence Diagram

위 절차 개요를 좀더 상세하게 시퀀스 다이어그램으로 표현해보면 다음과 같다.

![Imgur](https://i.imgur.com/LbGTeGn.png)

(http://commandlinefanatic.com/cgi-bin/showarticle.cgi?article=art014 내용 참고하여 재구성)

초록색 화살표는 브라우저와 웹 서버의 통신을 나타내며, 파란색 화살표는 HTTP API 호출을 나타낸다.

이제 시퀀스 다이어그램을 토대로 실제 구현해보자.

# OAuth 1.0a 구현 - Consumer

첫 번째 시나리오는 직접 구현한 Consumer를 통해 Service Provider인 트위터의 API를 사용해서 트위터에 글을 올리는 것이다. 

## 사전 조건

먼저 트위터에 내가 만들 Consumer 앱을 등록해야 한다. 참고로 OAuth 1.0a에서 Consumer라고 부르는 애플리케이션을 트위터에서는 트위터 앱(Twitter App)이라고 부른다.

Consumer 앱을 트위터에 등록하려면 먼저 트위터 개발자 계정이 있어야 한다. [트위터 개발자 포털](https://developer.twitter.com/en/docs/basics/developer-portal/overview)에서 개발자 계정을 신청할 수 있다.

개발자 계정 신청과 트위터 앱 등록 과정 설명은 아래의 화면 캡처로 대신한다.

### 트위터 개발자 계정 신청
 
![Imgur](https://i.imgur.com/OiGrOKX.png)

![Imgur](https://i.imgur.com/Lp1vUPB.png)

![Imgur](https://i.imgur.com/R1NUNqg.png)

![Imgur](https://i.imgur.com/hV1K0xX.png)

![Imgur](https://i.imgur.com/csco9Qs.png)

![Imgur](https://i.imgur.com/J6ltjT6.png)

![Imgur](https://i.imgur.com/ZgGGlFB.png)

![Imgur](https://i.imgur.com/Ap50ZcF.png)

![Imgur](https://i.imgur.com/AXkdALv.png)

![Imgur](https://i.imgur.com/9QjOwMn.png)

![Imgur](https://i.imgur.com/EJKOIXN.png)

### 트위터 앱 등록

![Imgur](https://i.imgur.com/3DWV3GB.png)

![Imgur](https://i.imgur.com/85q36Jq.png)

![Imgur](https://i.imgur.com/ikYSSYN.png)

![Imgur](https://i.imgur.com/4OyDBKp.png)

![Imgur](https://i.imgur.com/P8CcCqg.png)

![Imgur](https://i.imgur.com/SYsQSMF.png)


## Consumer 앱 개발

Consumer 앱은 OAuth 1.0a 흐름을 파악하는데 필요한 최소한의 기능만을 담아 간단하게 개발한다. 기능은 다음과 같다.

- 글을 쓸 수 있는 폼 화면
- 권한 부여 요청 전송 (시퀀스 다이어그램 2번)
- 서명 생성 기능
- 접근 토큰 요청 전송 (시퀀스 다이어그램 14번)
- 서명 생성 기능
- 트위터에 글 쓰기 (시퀀퀀스 다이어그램 20번)

편의상 스프링 부트로 개발하며, 프로젝트 생성 등의 자세한 과정은 생략한다.

OAuth 1.0a Spec인 [RFC-5849](https://tools.ietf.org/html/rfc5849)를 따라 Consumer가 갖춰야 할 기능을 구현해보자. 전체 소스 코드는 [여기](https://github.com/HomoEfficio/scratchpad-oauth10a-consumer)에 있다.

### 프로젝트 생성

스프링 이니셜라이저에서 다음과 같이 최소한의 starter만 선택해서 프로젝트를 생성한다.

![Imgur](https://i.imgur.com/DcmkDvi.png)


### 글 쓰는 폼 화면

글 쓰는 폼 화면도 최대한 단순하게 구성했다.

![Imgur](https://i.imgur.com/xEk7x8p.png)

User가 '트위터에 남기기' 버튼을 클릭하면, Consumer 앱이 트위터에 Request Token 발급을 요청한다. 이 부분부터 자세히 살펴보자.


## Request Token 발급 요청

Request Token 발급 요청 내용은 스펙의 [2.1 Temporary Credentials](https://tools.ietf.org/html/rfc5849#section-2.1)에 나와있다. Service Provider에게 전송해야할 정보는 다음과 같다고 예시에 나와 있지만,

- `oauth_consumer_key`: Service Provider로부터 발급받은 Consumer key
- `oauth_signature_method`: 서명 방식. `HMAC-SHA1`, `RSA-SHA1`,`PLAINTEXT`의 3가지 방식이 있다.
- `oauth_callback`: Request Token 발급 후 Service Provider가 제공하는 권한 부여 화면에서 User가 Consumer에게 권한을 부여하면 리다이렉트 되는 Consumer의 callback API URI
- `oauth_signature`: 서명 값

실제로는 [3.1 Making Requests](https://tools.ietf.org/html/rfc5849#section-3.1)에 나온 것처럼 다음과 같은 정보도 함께 전송해야 한다.

- `oauth_token`: Request Token 발급 요청 시에는 `oauth_token`이 없으므로 생략 가능
- `oauth_timestamp`: 1970.01.01 00:00:00 기준 요청 당시의 초 값
- `oauth_nonce`: 임의의 문자열 값으로 replay attack을 막는데 사용되며, timestamp, consumer key/secret과 request/access token이 같은 요청에 대해서 nonce 값은 유일해야 한다.
- `oauth_version`: 선택 사항이며 `1.0`이어야 한다.

이 규약은 Request Token 발급 요청 뿐아니라 Access Token 발급 요청 시에도 마찬가지로 적용된다.

대부분 이미 정해져 있거나 임의의 값 등으로 쉽게 구할 수 있지만, `oauth_signature`는 스펙에 정해진 규칙에 따라 계산 로직을 구현해줘야 한다.

이 글에서는 3가지 서명 방식 중 `HMAC-SHA1`만 다룬다. `PLAINTEXT`는 서명 방식으로 분류하고 있지만 실제로는 서명 방식이 아니며, `RSA-SHA1`는 shared-secret 대신 공개키/비밀키를 사용한다는 점만 `HMAC-SHA1`와 다르다.


## Token Signature

토큰 발급 요청을 위한 서명 생성 방법은 [3.4 Signature](https://tools.ietf.org/html/rfc5849#section-3.4)에 나와있다.

요약하면 다음과 같다.

1. Signature Base String 생성

1. Signature Base String을 Base64로 인코딩 한 값을 Secret으로 서명

### Signature Base String 생성

Signature Base String 생성 방식은 코드로 보는 것이 이해하기 쉬울 것 같다.

```java
private String generateBaseString(AbstractOAuthRequestHeader header) {
    String httpMethod = header.getHttpMethod();
    String baseUri = getBaseStringUri(header);
    String requestParameters = getRequestParameters(header);

    final StringBuilder sb = new StringBuilder();
    sb.append(httpMethod)
            .append('&').append(getUrlEncoded(baseUri))
            .append('&').append(getUrlEncoded(requestParameters));

    return sb.toString();
}
```

요약하면 Signature Base String은 HTTP 메서드, Base String URI(Token 발급 요청 URI), Token 발급 요청 파라미터를 [Percent encoding](https://tools.ietf.org/html/rfc5849#section-3.6) 한 후 &를 구분자로 이어 붙여서 만든다.

여기서 주의할 것은 Java의 `URLEncoder.encode`는 OAuth 1.0a 스펙에서 말하는 Percent encoding과 차이가 있어서 다음과 같이 보완해줘야 한다(이 부분을 간과해서 많은 시간 삽질을 해야했다 ㅠㅜ).

```java
public static String getUrlEncoded(String value) {
    try {
        return URLEncoder.encode(value, StandardCharsets.UTF_8.name())
                .replaceAll("\\+", "%20")
                .replaceAll("%21", "!")
                .replaceAll("%27", "'")
                .replaceAll("%28", "(")
                .replaceAll("%29", ")")
                .replaceAll("%7E", "~");
    } catch (UnsupportedEncodingException e) {
        throw new RuntimeException(e);
    }
}
```

이 중에서 Token 발급 요청 파라미터를 구성하는 방식은 더 자세히 살펴봐야 한다.

### Token 발급 요청 파라미터

Token 발급 요청 파라미터는 [3.4.1.3.  Request Parameters](https://tools.ietf.org/html/rfc5849#section-3.4.1.3)에 나와있다. 요약하면 다음과 같다.

1. Token 발급 요청 URI에 있는 Query String을 이름/값으로 파싱하고 URL decoding 한다.
1. Authorization 헤더에 있는 헤더 정보를 이름/값으로 파싱하고 URL decoding 한다.
1. 발급 요청이 single-part 이고 `Content-Type` 헤더 값이 `application/x-www-form-urlencoded`라면 HTTP 요청 body 값을 이름/값으로 파싱하고 URL decoding 한다.
1. 위의 값들을 normalization 한다. normalization 방식은 다음과 같다.
   1. 파라미터 이름과 값을 URL encoding 한다.
   1. 파라미터를 이름 기준 오름차순으로 정렬한다. 이름이 동일할 경우 값 기준 오름차순으로 정렬한다.
   1. 파라미터 이름과 값을 `=`로 이어 붙인다.
   1. 이어 붙인 파라미터를 `&`로 이어 붙인다.

스펙에서는 고맙게도 이에 대한 테스트 케이스를 제공해주는데 아래와 같은 토큰 발급 요청이 있다면,

```
POST /request?b5=%3D%253D&a3=a&c%40=&a2=r%20b HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Authorization: OAuth realm="Example",
              oauth_consumer_key="9djdj82h48djs9d2",
              oauth_token="kkk9d7dh3k39sjv7",
              oauth_signature_method="HMAC-SHA1",
              oauth_timestamp="137131201",
              oauth_nonce="7d8f3e4a",
              oauth_signature="djosJKDKJSD8743243%2Fjdk33klY%3D"

c2&a3=2+q
```

토큰 발급 요청 파라미터는 다음의 결과값을 갖게 된다.
(가독성을 위해 줄바꿈을 적용했으나 그런다고 가독성이 좋아지지는..)

```
a2=r%20b&a3=2%20q&a3=a&b5=%3D%253D&c%40=&c2=&oauth_consumer_key=9dj
dj82h48djs9d2&oauth_nonce=7d8f3e4a&oauth_signature_method=HMAC-SHA1
&oauth_timestamp=137131201&oauth_token=kkk9d7dh3k39sjv7
```

토큰 발급 요청 파라미터를 구하는 로직을 스펙을 읽고 정확하게 파악하는 것이 어렵지만, 일단 파악하면 구현 자체는 어렵지 않다. 필요하다면 [이걸](https://github.com/HomoEfficio/scratchpad-oauth10a-consumer/blob/master/src/main/java/io/homo/efficio/scratchpad/oauth10a/consumer/util/OAuth10aSupport.java) 참고하면 된다.


### 서명 생성

서명 생성은 [3.4.2.  HMAC-SHA1](https://tools.ietf.org/html/rfc5849#section-3.4.2)에 나와있다. 서명에는 키와 데이터가 필요한데 `HMAC-SHA1` 방식의 키는 Consumer Secret과 Token Secret을 `&`로 이어 붙인 값이다.

Request Token 발급 요청할 때는 Token Secret이 없는 상태이므로 그냥 `ConsumerSecret값&`이 키가 된다.

서명에 사용될 데이터는 위에서 구한 토큰 발급 요청 파라미터다.

서명 값은 `javax.crypto.Mac` 클래스를 이용해서 계산할 수 있으며, 검색해보면 찾을 수 있다.

```java
public void fillSignature(AbstractOAuthRequestHeader header) {
    String key = header.getKey();
    String baseString = generateBaseString(header);
    try {
        final SecretKeySpec signingKey = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), OAuth10aConstants.HMAC_SHA1_ALGORITHM_NAME);
        final Mac mac = Mac.getInstance(OAuth10aConstants.HMAC_SHA1_ALGORITHM_NAME);
        mac.init(signingKey);
        final String signature = Base64.getEncoder().encodeToString(mac.doFinal(baseString.getBytes(StandardCharsets.UTF_8)));
        header.setOauthSignature(signature);
    } catch (NoSuchAlgorithmException e) {
        throw new RuntimeException(e);
    } catch (InvalidKeyException e) {
        throw new RuntimeException(e);
    }
}
```

### 괴로운 부분

결론만 보면 쉬운 것 같지만 **직접 구현한 로직으로 만든 서명이 실제로 맞는지 검증을 하는 쉬운 방법이 없다는 게 OAuth 1.0a Consumer를 구현하는 과정 중에 가장 괴로운 부분이다.** 

서명이 맞는지 확인 하는 유일한 방법은 Service Provider인 트위터에 서명을 전송하고 트위터의 응답을 받아보는 것 밖에 없다. 그런데 서명이 맞지 않을 때는 400 Bad Request 만 확인할 수 있을 뿐이고, 디버거를 활용해서 확인해보면 아래와 같이 트위터가 알려주는 정보를 확인할 수는 있는데,

![Imgur](https://i.imgur.com/4qd0vlF.png)

해당 에러 코드의 내용을 [트위터 응답 코드 문서](https://developer.twitter.com/en/docs/basics/response-codes.html)에서 찾아봐도 아래와 같이 인증 정보가 없거나 잘못되었다는 그다지 건더기 없는 말만 볼 수 있을 뿐이다. 실로 막막하고 괴롭다.

![Imgur](https://i.imgur.com/o5SpdHs.png)

서명 검증이라는 것이 입력값을 하나하나 검증한다기보다 해시 계산 결과 일치 여부만으로 판단하므로 사실 구체적으로 어느 부분이 틀렸는지 알 수 없는 것이 당연하다.

그나마 할 수 있는 방법은 다음과 같다.

- 잘 동작하는 서명 생성 구현체를 찾아서 그 구현체로 구한 서명과 내 서명을 비교하고,
- 결과값이 다르다면 로직이 잘못되었거나 파라미터 정보 어딘가에 오타가 있다는 얘기이므로,
- 꼼꼼하게 뒤져서 바로잡는 수 밖에 없다.

그래도 여전히 막막하고 괴롭기는 마찬가지다.

여기까지 Request Token 발급 요청과 Access Token 발급 요청을 위한 서명 생성까지 다뤘다. 실제 화면으로 작업 흐름을 되짚어 보고 Access Token 발급까지 확인해보자.

### User의 권한 부여 신청

![Imgur](https://i.imgur.com/ROuYV2h.png)

### Consumer가 서명 생성 후 Service Provider에 전송해서 Request Token을 발급 받고, Service Provider가 제공하는 User의 권한 부여 화면으로 리다이렉트

![Imgur](https://i.imgur.com/KVmocfW.png)

### User가 앱 인증을 클릭하면 Consumer의 callback API로 리다이렉트

![Imgur](https://i.imgur.com/BpTOODs.png)

### callback API에서 서명 생성 후 Service Provider에 전송해서 Access Token 발급

![Imgur](https://i.imgur.com/4eia8Qv.png)




