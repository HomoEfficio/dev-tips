# 스펙따라 만들어보는 OAuth 1.0a Server

이 글은 [스펙따라 만들어보는 Oauth 1.0a Client](https://github.com/HomoEfficio/dev-tips/blob/master/스펙따라%20만들어보는%20OAuth%201.0a%20Client.md)의 후속편으로, Twitter가 담당했던 OAuth 1.0a Service Provider 역할을 하는 서버를 스펙따라 만드는 과정을 다룬다.

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

**1.0의 용어가 더 구별하기 쉽고 직관적이어서 학습하기에 좋으므로 본 글에서는 1.0 용어를 사용한다.** 1.0 용어로 이해한 후에는 1.0a의 용어도 쉽게 받아들일 수 있을 것이다.

한 가지 짚고 넘어갈 용어로 Secret이 있는데, 서명 방식에서 **Secret은 Service Provider가 Consumer에게 발급하고 둘이 각자 보유하고 있다가 필요할 떄 사용할 뿐 온라인으로 주고 받지 않는 정보**다.

## OAuth 1.0a Sequence Diagram

![Imgur](https://i.imgur.com/quqloI2.png)

(http://commandlinefanatic.com/cgi-bin/showarticle.cgi?article=art014 내용 참고하여 재구성)

초록색 화살표는 브라우저와 웹 서버의 통신을 나타내며, 파란색 화살표는 HTTP API 호출을 나타낸다.

시퀀스 다이어그램에 따르면 OAuth 1.0a 서버(서비스 프로바이더)가 갖춰야 할 기능은 다음과 같다.

1. Consumer Key, Consumer Secret 발급 및 저장

1. Request Token 발급 및 저장

   1. 권한 부여 신청 요청에 포함된 서명 검증
   1. Request Token 발급 및 저장

1. 사용자가 Consumer에게 권한을 부여할 수 있는 기능 제공

   1. 사용자 로그인
   1. 사용자 권한 부여 화면
   1. 권한 부여 확인값(`oauth_verifier`) 발급 및 저장

1. Access Token 발급 및 저장

   1. Access Token 발급 요청에 포함된 서명 검증
   1. Request Token 삭제
   1. Access Token 발급 및 저장

1. 보호 자원 제공

   1. 보호 자원 제공 요청에 포함된 서명 검증
   1. 요청 받은 보호 자원 제공

먼저 프로젝트 생성부터 시작해보자. OAuth 1.0a 관련 부분만 상세히 다루고 나머지 부분은 짚고 넘어가는 정도로만 다룬다.

## 프로젝트 생성

여러 방법으로 서버를 만들 수 있지만 이 글에서는 다음 환경으로 서버를 만든다.

- Java 9
- Gradle 4.8.1
- SpringBoot 2.0.4
- Spring Data JPA
- FreeMarker

OAuth 1.0a 이 아닌 나머지 부분은 간단함을 유지하기 위해 FreeMarker를 사용해서 서버 사이드 렌더링으로 구현한다.

스프링 이니셜라이저에서 다음과 같이 선택하고 프로젝트를 생성한다.

![Imgur](https://i.imgur.com/kkt23AA.png)

Java 9에서는 [JAXB 관련 에러가 발생](https://github.com/HomoEfficio/dev-tips/blob/master/Java%209%20JAXBException.md)할 수 있으므로 jaxb-api 의존 관계를 추가한다.

SHA-256 해시 함수를 사용해야하므로 Apache Commons Codec 의존 관계도 추가한다.

Lombok의 의존 관계도 `compileOnly`에서 `compile`로 바꾼다.

결과적으로 build.gradle 파일에 구성되는 의존 관계는 다음과 같다.

```
dependencies {
    compile('org.springframework.boot:spring-boot-starter-data-jpa')
    compile('org.springframework.boot:spring-boot-starter-freemarker')
    compile('org.springframework.boot:spring-boot-starter-web')
    compile('org.springframework.boot:spring-boot-devtools')
    compile('javax.xml.bind:jaxb-api:2.3.0')
    compile('org.projectlombok:lombok')
    compile group: 'commons-codec', name: 'commons-codec', version: '1.11'

    runtime('com.h2database:h2')
    testCompile('org.springframework.boot:spring-boot-starter-test')
}
```


## Consumer Key, Consumer Secret 발급 및 저장

OAuth 1.0a 스펙은 Consumer Key, Consumer Secret 발급 및 저장에 대해 규정하고 있지 않다. 

시퀀스 다이어그램에도 나와있지만 Consumer Key, Consumer Secret은 OAuth 1.0a 를 통한 권한 부여 및 인증/인가 프로세스를 시작하기 전에 이미 정해져 있는 사전 조건이라고 볼 수 있다.

별도로 정해진 방법이 없으므로 식별성을 유지할 수 있다면 마음대로 정할 수 있다. 여기에서는 간단하게 다음 방법으로 생성한다.

- Consumer Key: Random UUID 값을 Base64 인코딩한 값

  ```java
  Base64.getEncoder().encodeToString(UUID.randomUUID().toString().getBytes());
  ```
- Consumer Secret: Random UUID 값을 SHA-256으로 해시한 후 Base64 인코딩한 값

  ```java
  Base64.getEncoder().encodeToString(DigestUtils.sha256Hex(UUID.randomUUID().toString().getBytes()).getBytes());
  ```

### Consumer 도메인 객체

Consumer 정보를 저장할 도메인 객체는 간단하게 id, name, description, callbackUrl, consumerKey, consumerSecret만으로 구성한다.

```java
@Entity
@Table(name = "consumer")
@Data
public class Consumer extends BaseEntity {

    @Id
    @GeneratedValue
    @Column(name = "consumer_id")
    private Long id;

    @NotNull
    private String name;

    @NotNull
    private String description;

    @NotNull
    private String callbackUrl;

    private String consumerKey;

    private String consumerSecret;


    public Consumer(@NonNull String name, @NonNull String description, @NonNull String callbackUrl) {
        this.name = name;
        this.description = description;
        this.callbackUrl = callbackUrl;
    }
}
```

`@Data`를 붙여서 결과적으로 모든 필드에 대해 `@Setter`를 생성하는 것은 권장할만한 코딩 방식은 아니지만 OAuth 1.0a 외의 부분은 최대한 간단하게 작성하기 위해 `@Data`를 사용한다.

### Consumer 등록 화면

FreeMarker로 다음과 같이 간단하게 작성한다.

![Imgur](https://i.imgur.com/BNegUK2.png)

### Consumer 등록 결과 화면

앱을 등록하면 다음과 같이 발급된 Consumer Key, Consumer Secret과 등록된 Callback URL을 보여준다.

![Imgur](https://i.imgur.com/AvREvfg.png)


이어서 작성..


