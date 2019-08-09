# POSTMAN - POST로 보냈는데 GET 요청으로 인식

## 현상

POSTMAN을 이용해서 분명히 POST로 보낸 요청인데 API 서버에서 GET 메서드로 인식되는 문제가 있었다.

![Imgur](https://i.imgur.com/Wc9W7pm.png)

## 원인

WAS 앞단에 있는 Web 서버 로그를 보니 이유를 알 것 같다.

![Imgur](https://i.imgur.com/xK2AE0g.png)

Web 서버는 301 Moved Permanently 를 반환했고, POSTMAN이 301에 따라 리다이렉트 하면서 GET으로 요청을 다시 보내기 때문이다.

왜 POST로 보내지 않고 GET으로 보내나 찾아보니 https://support.getpostman.com/hc/en-us/articles/211913929-I-sent-a-POST-request-but-Postman-seems-to-be-sending-a-GET-request- 이런 자료가 나왔다.

>This is mostly due to a server-side redirect. If Postman receives a 301 or 302 response code, it will automatically follow the redirect. Similar to the behaviour in browsers, the original request method will NOT be preserved, and **the redirected request will be made with a GET method**.

## 부가 정보

HTTP Redirection 관련해서 [MDN 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Redirections)를 보면, 다음과 같은 내용이 있다.

https://developer.mozilla.org/en-US/docs/Web/HTTP/Redirections#Permanent_redirections

요약하면 다음과 같다.
>301은 GET이 아닌 메서드는 GET으로 바뀌어 리다이렉트 될 수도 있고 원래 메서드 그대로 리다이렉트 될 수도 있다.  
>308은 메서드와 바디 모두 그대로 유지한채로 리다이렉트 되어야 한다.

참고로 [HTTP 308 스펙은 여기](https://tools.ietf.org/html/rfc7538)에 있으니 나중에 보고 싶으면 보는 걸로 ㅋ  
최종판이 2015년에 나왔으니 비교적 최근에 추가된 코드라고 할 수 있겠다.

## 해결

- 웹서버(nginx)에서 리다이렉트 시 301 대신 308을 회신하도록 설정하면 원론적으론 해결될 것 같다. 그런데 클라이언트가 그에 맞게 구현했는지는 까봐야 알 수 있는 상황.
- 예상대로 POSTMAN은 308을 회신 받아도 GET으로 바꿔서 전송한다.
  - 관련 이슈가 있었지만 https://github.com/postmanlabs/postman-app-support/issues/4114, https://github.com/postmanlabs/postman-app-support/issues/5599 어쨌든 아직도 308에도 GET으로 바꿔 전송한다.
- POSTMAN에서는 그냥 https로 호출하는 수 밖에 없고, 웹 브라우저는 308에 어떻게 반응하는지 확인 후 웹 서버 설정 적용
