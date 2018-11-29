# XHR 요청 시 canceled

다음과 같이 XHR 요청을 위한 OPTIONS 는 200으로 잘 실행됐는데 본 요청에서 (canceled)로 표시될 때가 있다.

![Imgur](https://i.imgur.com/2ZfmIZE.png)

이는 `form` 이 있는 화면에서 `e.preventDefault()` 를 해주지 않아서 기본 `submit()` 이 실행되기 때문에 발생하는 문제다.

따라서 요청을 보내는 이벤트 핸들러에서 `e.preventDefault()` 를 해줘야 한다.

VueJS는 이벤트 수식어를 사용해서 `@click.prevent='login'`와 같이 해줘도 된다.

