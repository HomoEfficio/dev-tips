# img 태그를 이용한 정보 전달

현재 웹 페이지 로딩 중에 다른 도메인에 정보를 전달해야할 때가 있다. 

특히 반환 받을 필요 없이 일방적으로 정보를 보내기만 하면 되는 상황이라면 AJAX를 사용하지 않고 단순히 `<img src='정보를 보낼 곳?보낼정보항목=보낼정보값&...'>` 와 같이 `img` 태그만으르도 정보를 보낼 수 있다.

그런데 브라우저마다(특히 IE) `img` 태그를 처리하는 방식이 다를 수 있으므로 주의해야 한다.

크롬에서 아래와 같이 작성하면, 이미지를 렌더링하거나 DOM에 추가하지 않고 `src`에 명시된 곳으로 요청을 보내기만 한다. 즉, 의도하는 대로 동작한다.

```javascript
let pixel = document.createElement('img');
pixel.src = '정보를 보낼 곳?보낼정보항목=보낼정보값&...';
document.body.appendChild(pixel);
```

하지만 IE에서 위와 같이 처리하면 엑스박스가 나와서 화면 UI를 해치게 된다. 그래서 아래와 같이 `width`와 `height`를 0으로 명시해주면, 엑스박스는 안 나오지만 화면에서 눈에 띌 만한 일부 영역을 공백으로 차지하면서 여전히 UI를 깨뜨린다.

```javascript
let pixel = document.createElement('img');
pixel.src = '정보를 보낼 곳?보낼정보항목=보낼정보값&...';
pixel.widgh = 0;
pixel.width = 0;
document.body.appendChild(pixel);
```

어떻게 하면 좋을까?

아래와 같이 `document.body.appendChild(pixel);`을 제거해서 DOM에 추가하지 않으면 된다.

```javascript
let pixel = document.createElement('img');
pixel.src = '정보를 보낼 곳?보낼정보항목=보낼정보값&...';
pixel.widgh = 0;
pixel.width = 0;
```

이렇게 하면 DOM에 추가가 안 되므로 `src`로 명시한 URL에 요청을 안 보낼 것 같지만, 개발자 도구의 Network 탭에서 확인해보면 요청은 보낸다.

또는 더 깔쌈한 해결 방식은

>(new Image()).src = '정보를 보낼 곳?보낼정보항목=보낼정보값&...'

와 같이 `new Image()`를 사용하면 `src`만 지정해줘도 크롬뿐만아니라 IE에서도 의도대로 동작한다.

이렇게 `img` 태그로 정보를 전달하는 방법은 광고 업계에서 사용자를 식별하는 데 사용되는 쿠키 싱크(https://clearcode.cc/blog/cookie-syncing/)에서 주로 사용 된다.
