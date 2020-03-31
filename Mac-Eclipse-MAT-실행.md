# Mac Eclipse MAT 실행

힙 덤프 파일을 열어서 분석해볼 수 있는 [Eclipse MAT](https://www.eclipse.org/mat/) 라는 도구가 있다.

예상되는 문제 지점을 스스로 분석해서 보고서로 보여주는 기능이 있어서, VisualVM 과 더불어 힙 덤프 분석에 널리 사용된다.

맥에서는 일단 다운로드 받고 압축 풀어 실행하면 '확인되지 않은 개발자..' 어쩌구 하면서 실행 자체가 안 된다.  
이 때는 Command 인지 Control 인지와 함께 클릭 후 열기 를 누르고 어쩌구 확인 누르면 실행할 수 있다.

그런데 실행하면 다음과 같은 에러와 함께 실행이 안 되기도 한다.

![Imgur](https://i.imgur.com/UOynHoo.png)

Eclipse MAT 도 어차피 Eclipse 니까 어딘가에 있을 eclipse.ini 을 찾아서 뭔가 수정해주면 될텐데, 맥에서는 달랑 mat 파일 하나만 있어서 당황스럽다.


## ini 파일 찾기

일단 다운로드 받고 압축 해제한 mat 파일을 `응용 프로그램` 폴더로 이동시킨다.

응용 프로그램 폴더에서 mat 파일을 우클릭하면 `패키지 내용 보기` 메뉴가 나온다 이를 클릭하면 다음과 같이 Contents 폴더가 있고,

![Imgur](https://i.imgur.com/lKmmjfU.png)

Contents > Eclipse 로 들어가면 다음과 같이 드디어 반가운 ini 파일을 볼 수 있다.

![Imgur](https://i.imgur.com/9kxbLZy.png)

## vm 옵션 지정

다음과 같이 vm 옵션으로 java 실행 파일의 경로를 직접 지정해준다.

![Imgur](https://i.imgur.com/sJocMmm.png)

## 실행

다시 응용 프로그램 폴더에서 mat 를 더블 클릭하면 몇 초 후에 다음과 같이 드디어 정상적으로 실행된다.  
(와 Eclipse 화면 오랜만에 본다..)

![Imgur](https://i.imgur.com/k7o679h.png)

혹시 이렇게 해도 실행 시 오류가 나면 몇 초 기다렸다가 다시 실행하면 되더라능..





