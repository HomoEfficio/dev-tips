# JSONP, Content-Type, X-Content-Options:nosniff

## JSONP

브라우저의 Same Origin Policy를 우회하는 방법 중의 하나로 JSONP(JSON with Padding)가 있다. JSONP는 `<script>` 요소의 `src` 속성으로 가져오는 resource는 Same Origin Policy의 제약을 받지 않는다는 점을 이용한다. 코드로 보자면 이런 식이다.

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

원칙적으로는 `http://ANY_SERVER/jsonp-test?callback=printToScreen`라는 자원을 가지고 있는 서버가 값을 반환할 때 `Content-Type`을 `text/javascript`나 `applicatino/javascript` 등 실행될 수 있는 타입으로 지정해줘야만, 결과값을 받는 클라이언트에서 `printToScreen("HELLO JSONP");`라는 문자열을 JavaScript로서 실행할 수 있다.

그런데 실제로는 서버가 `Content-Type`을 명확히 지정해주지 않아도 똑똑한 브라우저가 알아서 판단해서 `printToScreen("HELLO JSONP");`라는 문자열을 JavaScript로서 실행해준다.

## X-Content-Options





