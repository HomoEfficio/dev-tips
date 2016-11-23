# (망할)Mac에서 git push 하다 에러나면..

`ssh-key` 등 GitHub에 등록 다 하고 `git push`를 해보니 아래와 같은 에러가 난다.

>remote: Permission to HomoEfficio/spring-web-api-test-stubber.git denied to hanmomhanda.

## 원인

에러 화면을 보니 git repository에서 설정한 `user.name`과 다른 id로 push를 시도하고 있는 것으로 보인다.

![Imgur](http://i.imgur.com/dl4UeYp.png)

한참 검색하고 찾아보다가 왜 그런지 답을 찾았다.

http://stackoverflow.com/a/31005094

원인은 역시나.. 또 (망할)Mac이다..

## 해결

해결 방법은 다음과 같다.

1. `Alfred`에서 `Access`라고 입력하면 `키체인 접근`이라는 앱이 나온다. 실행시킨다.
2. 검색어에 `git`를 입력하면 다음과 같이 필터링 된 결과가 나온다.

	![Imgur](http://i.imgur.com/SAY4oAT.png)
	
3. 하나씩 클릭해서 의심되는 것(내 경우에는 hanmomhanda라는 정보가 포함된 것)을 지운다. 지워도 `git` 쓰는 데는 문제 없다.

지우고 나니 아래와 같이 잘 동작한다.

![Imgur](http://i.imgur.com/9hxyZsc.png)

## 정리

>- 난 Mac이 싫다. ㅋ
