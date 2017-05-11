# Referer와 403 Forbidden

외부 API를 이용하다보면 분명히 API 인증 정보 등이 정확함에도 불구하고, 요청에 대해 403 Forbidden을 만나게 되는 경우가 있다.

신기하게도 요청 URL을 브라우저에 그대로 입력하면 403 Forbidden 없이 정상적으로 응답을 받지만, 애플리케이션 내에서 동일한 리소스를 요청하면 403 Forbidden이 발생하기도 한다.

오늘도 그런 현상을 만나서 Request Header를 유심히 관찰하다보니, `Referer` 헤더의 유무에 따라 403 Forbidden 발생 여부가 갈리고 있었다. 
즉, 헤더에 `Referer`가 없으면 정상 회신, `Referer`가 있으면 403 이었다.

그래서 어떻게 `Referer`가 포함되지 않게 할 수 있나 검색을 해보니 [이걸](http://stackoverflow.com/questions/6817595/remove-http-referer/32014225#32014225) 찾아냈는데, 가장 단순하게는 다음과 같이 meta 태그로 해결할 수 있다.

>`<meta name="referrer" content="never" />`

물론 저 meta 태그 정보를 브라우저가 지원하지 않을 수도 있고(http://caniuse.com/#feat=referrer-policy 참고), `Referer`가 없는 요청을 받아들이지 않는 서버도 있을 것이므로 어느 정도 편법에 불과하다고 볼 수 있다.
