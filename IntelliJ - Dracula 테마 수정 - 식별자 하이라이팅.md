# IntelliJ - Dracula 테마 수정 - 식별자 하이라이팅

팔랑귀인 나는 누군가 `Dracula` 테마가 좋다는 얘기를 듣고 말았다.

그래서 https://draculatheme.com/jetbrains/ 여기에 들어가서 테마를 기존의 IntelliJ 디폴트 테마인 `Darcula`에서 `Dracula`로 바꿔 적용했다.

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

1. 아래와 같이 `Code > Identifier under caret`을 선택하고 우측에 있는 `Background`의 색을 적당한 색으로 바꿔준다.

	![Imgur](http://i.imgur.com/uDLvfqd.png)

1. `Code > Identifier under caret(write)`를 선택하고 우측에 있는 `Background`의 색을 적당한 색으로 바꿔준다. 이렇게 하면 **할당되어 값이 변할 가능성이 있는 식별자는 다른색으로 하이라이팅** 되어 코드 분석할 때 도움이 된다.

	![Imgur](http://i.imgur.com/0zHsJf6.png)

이제 결과를 보자.

## Dracula 테마 수정 결과

감동이다.. 저용량 고휘발성 메모리 보유 지진개발자의 까막눈가에 흐르는 눈물을 닦아본다(내친김에 폰트도 `Menlo`로 바꿨다).

![Imgur](http://i.imgur.com/5B1hSpE.png)
