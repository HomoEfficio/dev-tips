# IntelliJ - 에디터 테마 수정 - 식별자 하이라이팅

## IntelliJ - 에디터 테마 적용 방법

에디터 테마 적용 방법은 다음과 같다.

1. themeX.icls 파일을 특정 폴더에 저장하면,
1. 설정에서 themeX를 선택할 수 있다.

윈도우에서의 그 특정 폴더의 위치를 찾는 방법은 다음과 같다.

- File > Import Settings... 를 클릭하면,

![](https://i.imgur.com/Uy2SL55.png)

- 아래와 같이 기본으로는 config 폴더가 선택되어 나오는데, 그 바로 아래의 color 폴더에 icls 파일을 저장하면 된다.

![](https://i.imgur.com/rnjmavL.png)

- IntelliJ를 재시작 하면 Dracula 테마를 선택할 수 있다.

![](https://i.imgur.com/W5tEmcQ.png)

## Dracula 테마

팔랑귀인 나는 누군가 `Dracula` 테마가 좋다는 얘기를 듣고 말았다.

그래서 https://draculatheme.com/jetbrains/ 여기에 들어가서 테마를 기존의 IntelliJ 디폴트 테마인 `Darcula`에서 `Dracula`로 바꿔 적용했다.

사이트에 맥의 적용 방법은 나와있는데 

다른 언어는 어떨지 모르겠는데 Java는.. 솔직히 좀.. 내 취향은 아니다.. ㅠㅜ

![Imgur](http://i.imgur.com/PzrfkR3.png)

뭐 이런데 민감한 편이 아니라 걍 써보지 뭐.. 하는데 도저히 그냥 지나칠 수 없는 것이 있었으니..

![Imgur](http://i.imgur.com/mYb62BG.png)

뭐지 이 괴랄하고 아스트랄한 하이라이팅은..

배경색이 검은색인데 커서가 있는 식별자의 배경색을 똑같이 검은색으로 하이라이팅 해준다?? 

지금이야 커서 있는 행이 다른 색으로 하이라이팅 되고 있어서 검은색 하이라이팅이 괴랄아스트랄하게나마 티가 나지만 **커서 없는 행에 있는 식별자는 사실 상 하이라이팅이 안 된다는거자나!!**

이래서는 안된다..
특히 한 줄 코딩하고 난 직후에도 바로 위에 줄에 무슨 코딩을 했는지 기억을 못하는 저용량 고휘발성 메모리를 보유한 나에게는.. 이따위 하이라이팅은 그냥 메모장과 다를 바 없다..

바꾸자..

## 식별자 하이라이팅 변경

1. 가급적 원본 보존의 법칙을 지켜주는거다. `Preferences > Editor > Colors & Fonts > General` 에 들어가서 `Scheme`을 복사해서 새로 하나 만든다.

    ![Imgur](http://i.imgur.com/Y0aJWFb.png)

1. 아래와 같이 `Code > Identifier under caret`을 선택하고 우측에 있는 `Background`의 색을 적당한 색(ex: 475D55)로 바꿔준다.

    ![Imgur](http://i.imgur.com/uDLvfqd.png)

1. `Code > Identifier under caret(write)`를 선택하고 우측에 있는 `Background`의 색을 적당한 색(ex: 721A20)으로 바꿔준다. 이렇게 하면 **할당되어 값이 변할 가능성이 있는 식별자는 다른색으로 하이라이팅** 되어 코드 분석할 때 도움이 된다.

    ![Imgur](http://i.imgur.com/0zHsJf6.png)

이제 결과를 보자.

## 주석 글자색 변경

Dracula의 주석 글자색은 좀 어두운 편이라서 조금 밝게 해주면 좋다. 다음과 같이 Editor > Colors & Fonts > Java 의 Comments에서 Block comment, JavaDoc>Text, Line comment 의 Foreground 색을 적당한 값(ex: B6C8FF)으로 바꿔준다.

![Imgur](http://i.imgur.com/MkBKMVL.png)


## Dracula 테마 수정 결과

감동이다.. 저용량 고휘발성 메모리 보유 지진개발자의 까막눈가에 흐르는 눈물을 닦아본다(내친김에 폰트도 `Menlo`로 바꿨다).

![Imgur](http://i.imgur.com/5B1hSpE.png)


## Ladies Night 2 테마

http://color-themes.com/ 에는 좋은 테마가 많다. 그 중에서 가장 인기 있는 Ladies Night 2를 적용해보니, 괜찮다!

![](https://i.imgur.com/BJI5L5M.png)

클릭하면 jar 파일을 다운로드 받게 되는데, File > Import Settings... 에서 다운로드 받은 jar 파일을 선택하고 재시작 하면 다음과 같이 설정에서 테마를 선택할 수 있다.

![](https://i.imgur.com/0XJhfFj.png)

위에 식별자 하이라이팅에 나온 것에서 `Code > Identifier under caret(write)`에 대한 값만 `721A20`로 해주면 아주 좋다!!

![](https://i.imgur.com/c127Fb8.png)


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.

