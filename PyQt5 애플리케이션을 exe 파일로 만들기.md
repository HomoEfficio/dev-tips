# PyQt5 애플리케이션을 exe 파일로 만들기

리눅스나 맥에서 PyQt5로 작성한 GUI 애플리케이션을 윈도우에서 실행되는 exe 파일로 만들어 보자.

윈도우에 파이썬은 설치되어 있다고 가정한다.

## pyinstaller 설치

명령 프롬프트를 관리자 모드로 실행한 후 다음 명령으로 `pyinstaller` 설치

>pip install pyinstaller

![Imgur](https://i.imgur.com/j1g9Y61.png)

## PyQt5 설치

다음 명령으로 PyQt5를 설치하자. 약 80메가 정도 된다.

>pip install PyQt5

![Imgur](https://i.imgur.com/r2NDpc5.png)

## PATH 설정

위의 설치화면에 나온대로 `c:\program files\python36\Scripts`를 PATH에 추가한다.

## exe 파일 하나로 생성

기본 옵션으로 실행하면 dist 폴더가 생기고 그 아래에 PyQt5 등 여러 라이브러리가 담긴 폴더가 생성되고, 그 안에 exe 파일이 생성되는데 이 파일을 실행하면 오류가 발생한다.

대신에 exe 파일 하나로 생성하는 -F 옵션을 주고 다음과 같이 실행하면 dist 폴더에 exe가 생성되는데, 이 파일은 제대로 실행된다.

>pyinstaller -F dmpFileSplitter.py

![Imgur](https://i.imgur.com/gQ1Ljer.png)

실행 과정에서 WARNING이 상당히 많이 뜨는데 다행히 생성되는 실행 파일에는 문제가 없다.

아래와 같이 dist 폴더 아래에 exe 파일이 생성된다.

![Imgur](https://i.imgur.com/uq8Ez4J.png)

## 실행

실행해보면 다음과 같이 콘솔 창 위에 GUI 애플리케이션이 실행된다.

![Imgur](https://i.imgur.com/lxsz6DT.png)

실제 파이썬 코드는 150라인 정도로 단순한 파일 나누기 프로그램이지만 PyQt5 라이브러리 등이 필요할테니 아무래도 실행 파일 크기가 커진다. 

무려 16메가가 넘는다. 실행 속도도 꽤 느린 편이다. 역시 윈도우용 GUI 프로그램은 윈도우에서 윈도우 라이브러리로 만들어야..
