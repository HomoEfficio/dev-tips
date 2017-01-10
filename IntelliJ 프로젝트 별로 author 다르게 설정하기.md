# IntelliJ 프로젝트 별로 author 다르게 설정하기

IntelliJ에서 새 파일 생성 시 자동으로 생성되는 Header에는 `author` 정보가 담겨 있는데, `author`를 프로젝트 별로 다르게 설정하고 싶다.

답은 여기에 있다.

http://stackoverflow.com/a/31936146

조금 보완해서 옮겨 보면

>1. `author`가 다르게 구성되길 원하는 프로젝트가 활성화 된 상태에서
>1. Preferences > Settings > Editor > File and Code Templates > Includes > File Header 에 들어가서
>1. 우상단의 `Schema`를 `Default`에서 `Project`로 변경
>1. 주석에서 원래 있던 `${USER}`를 원하는 `author`로 변경한다.

![Imgur](http://i.imgur.com/QZRwRvq.png)


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
