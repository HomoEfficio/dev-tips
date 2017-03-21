# JSONP, Content-Type, X-Content-Type-Options

## JSONP

브라우저의 Same Origin Policy를 우회하는 방법 중의 하나로 JSONP(JSON with Padding)가 있다. 

JSONP는 `<script>` 요소의 `src` 속성으로 가져오는 resource는 Same Origin Policy의 제약을 받지 않는다는 점을 이용한다. 

코드로 보자면 이런 식이다.

```html
<html>
<body>
<div id="div1"></div>
<script>
var jsonpResultScript = document.createElement('script');
jsonpResultScript.src = 'http://ANY_SERVER/jsonp-test?callback=printToScreen';  // callback으로 지정한 값 - (1)
jsonpResultScript.type = 'text/javascript'; - (2)
document.body.append(jsonpResultScript);
</script>
<script>
var printToScreen =  // (1)에서 지정한 값과 같은 printToScreen
    function(result) { 
        document.getElementById('div1').innerHTML = '<h1>' + result + '</h1>';
    };
</script>
</body>
</html>
```

`http://ANY_SERVER/jsonp-test?callback=printToScreen`라는 resource가 반환하는 값을 JavaScript 코드로서 실행한다는 것이다.

예를 들어, `http://ANY_SERVER/jsonp-test?callback=printToScreen`가 `printToScreen("HELLO JSONP");`라는 문자열을 반환하면 결국에는 아래와 같은 모양새가 되어,

```javascript
<script>
printToScreen("HELLO JSONP");
</script>
```

`printToScreen`이라는 JavaScript 함수를 실행하게 되는 것이다.

## Content-Type

그런데 `http://ANY_SERVER/jsonp-test?callback=printToScreen`라는 자원이 `printToScreen("HELLO JSONP");`라는 문자열을 반환한다고 해서 언제나 JavaScript로서 실행될 수 있는 것은 아니다.

원칙적으로는 `http://ANY_SERVER/jsonp-test?callback=printToScreen`라는 자원을 가지고 있는 서버가 값을 반환할 때 `Content-Type`을 `text/javascript`나 `application/javascript` 등 JavaScript로 실행될 수 있는 타입으로 지정해줘야만, 결과값을 받는 클라이언트에서 `printToScreen("HELLO JSONP");`라는 문자열을 JavaScript로서 실행할 수 있다.

그런데 실제로는 서버가 `Content-Type`을 명확히 지정해주지 않아도 똑똑한 브라우저가 알아서 판단해서 `printToScreen("HELLO JSONP");`라는 문자열을 JavaScript로서 실행해준다.

## X-Content-Type-Options: nosniff

하지만 위와 같이 resource의 소유자인 서버가 `Content-Type`를 지정해주지 않은 결과마저도 브라우저가 알아서 JavaScript로서 실행해버리면 보안 약점으로 악용될 수도 있다.

그래서 서버는 이처럼 브라우저가 마음대로 추측(sniff: 냄새로 무언가를 알아차리기)해서 JavaScript로서 실행하는 일을 강제로 못하게 할 수 있는 장치가 있는데, 바로 `X-Content-Type-Options: nosniff`라는 Response 헤더 값을 지정하는 것이다.

서버가 `Content-Type` 헤더 값을 `text/javascript` 등 JavaScript로서 실행할 수 있는 값으로 지정하지 않은 상태에서 `X-Content-Type-Options: nosniff` 헤더가 지정되면 아래와 같이 브라우저는 에러를 뱉어내게 된다.

** 브라우저 에러 캡쳐 **

이는 위에 있는 소스의 (2)에서 처럼 브라우저 쪽에서 아무리 `jsonpResultScript.type = 'text/javascript';`라고 지정해줘도 소용이 없고, 어디까지나 서버에서 지정한 `Content-Type` 값에 따라 에러 여부가 결정된다.

# 정리

>- JSONP를 사용한다면
>
>    - 서버에서는 보안을 위해 `X-Content-Type-Options: nosniff`를 꼭 지정해주자
> 
>        - Tomcat8에서는 `HttpHeaderSecurityFilter`라는 클래스를 통해 지정 가능
>        - Node.js + Express에서는 `helmet`라는 모듈을 통해 지정 가능
> 
>    - 서버에서는 `Content-Type`를 `text/javascript`로 꼭 지정해주자


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
