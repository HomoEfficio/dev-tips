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
