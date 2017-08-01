# Google Analytics(구글 어낼리틱스) 초간단 정리

구글 어낼리틱스는 웹 페이지의 **유입처 분석**을 가능하게 해준다. 사용자는 이 **유입처 분석**을 통해 사업 관점에서 다양한 통찰을 얻어낼 수 있다.

대략 아래 그림과 같은 정보를 제공해준다.

![Imgur](http://i.imgur.com/9yUBfOh.png)

## 유입처란?

www.initial.com 이라는 도메인 내의 어떤 페이지 A내에 있는 링크를 클릭해서 www.target.com 이라는 페이지로 B로 이동하면, B의 유입처는 www.initial.com 이 된다.

## 유입처를 어떻게 알 수 있나?

### Referer 헤더

www.initial.com의 페이지 A 안에 포함된 링크를 통해 www.target.com의 페이지 B로 이동할 때의 HTTP Request Header 중 `Referer`라는 항목에는 www.initial.com 이라는 값이 설정된다. 

>이 `Referer`라는 헤더 값에 유입처 정보가 담겨 있다.

웹 페이지 상의 링크를 통해 들어오지 않고 주소창에 직접 페이지 B의 URL을 입력(또는 북마크 등)해서 들어오면 `Referer` 헤더 자체가 HTTP Request 안에 들어있지 않다.

### document.referrer

www.target.com의 페이지 B로 이동한 후에 Console을 열어서 채`console.log(document.referrer);`로 확인해보면 `document.referrer`에는 www.initial.com의 페이지 A의 URL인 www.initial.com/pageA가 출력된다. 

>`document.referrer`에는 유입된 페이지 URL이 담겨있다.

웹 페이지 상의 링크를 통해 들어오지 않고 주소창에 직접 페이지 B의 URL을 입력(또는 북마크 등)해서 들어오면 `document.referrer`에는 빈 문자열이 들어있다.

참고로, HTTP의 헤더인 `Referer`에는 중간에 r이 한 개인데, `document.referrer`에는 중간에 r이 두 개다.

## 그럼 구글 어낼리틱스는 어떻게 사용?

쉽게 말해 구글 어낼리틱스에서 제공해주는 JavaScript를 유입처 분석을 원하는 모든 페이지에 적용하면 된다.

대략 아래와 같이 생겼다.

```html
<!-- Google Analytics -->
<script>
(function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
(i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
})(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

ga('create', 'UA-XXXXX-Y', 'auto');
ga('send', 'pageview');
</script>
<!-- End Google Analytics -->
```

위 코드는 대략 아래의 4가지 일을 수행한다.

1. `<script>` 태그를 만들고 `https://www.google-analytics.com/analytics.js` 파일을 비동기로 다운로드
1. 구글 어낼리틱스 기능을 사용할 수 있게 해주는 `ga` 함수(ga() 커맨드 큐라고 불리기도 한다)를 글로벌 스코프애 초기화
1. `ga('create', 'UA-XXXXX-Y', 'auto');`를 통해 새로운 tracker object를 생성한다. `UA-XXXXX-Y`는 구글 어낼리틱스가 고객에게 부여하는 식별자
1. `ga('send', 'pageview');`를 통해 `pageview`라는 이벤트를 구글 어낼리틱스에 전송

최초 설치 및 설정 등의 자세한 내용은 https://support.google.com/analytics 를 참고한다.

## 구글 어낼리틱스에게 유입처를 지정해 줄 수 있는 방법

보틍은 구글 어낼리틱스가 그냥 수동적으로 `Referer`나 `document.referrer`에 담겨 있는 값을 읽어서 유입처를 분석하지만, 적극적으로 구글 어낼리틱스에게 유입처를 지정해서 알려줄 수도 있다.

구글 어낼리틱스가 적용된 페이지에 들어올 때 URL Parameter에 `utm_source`라는 파라미터 값이 있으면, 구글 어낼리틱스는 그 값을 `Referer` HTTP 헤더나 `document.referrer` 값에 우선해서 해당 페이지의 유입처로 인식한다.

즉, 어떤 페이지 C의 `Referer` 헤더에 `www.initial.com`이 담겨 있더라도, 

페이지 C의 url이 www.target.com/pageC?utm_source=www.other.com 이라면,

구글 어낼리틱스는 페이지 C의 유입처를 `www.initial.com`이 아니라, `www.other.com`으로 인식한다.

구글 어낼리틱스에서는 이를 **맞춤 캠페인**이라는 이름으로 설명하고 있으며, 자세한 내용은 https://support.google.com/analytics/answer/1033863?hl=ko 를 참고한다.
