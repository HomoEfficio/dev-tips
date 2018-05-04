# 맥에서 Xcode 삭제 및 개발 환경 복원

디스크 용량이 없다고 해서, 거의 사용하지 않는데도 10G 이상의 용량을 차지하는 Xcode를 지우려고 한다.

## Xcode 삭제

깔끔하게 지우는 방법은 https://www.client9.com/uninstall-xcode-on-macos-sierra/ 에 잘 나와 있는데 요약하면 다음과 같다.

>1. sudo /Developer/Library/uninstall-devtools --mode=all
>
>    "not found" 어쩌구 나와도 걍 무시하자 어차피 지울거니까.

>2. 응용프로그램 폴더에서 Xcode를 휴지통으로 보낸다.

>3. 아래 세 명령으로 찌꺼기 파일을 지운다.
>
>    sudo rm -rf ~/Library/Developer/
>
>    sudo rm -rf ~/Library/Caches/com.apple.dt.Xcode
>
>    sudo rm -rf /Library/Developer/CommandLineTools

>3. make를 실행해서 뭔가 없다는 에러가 나오면 잘 지워진 것이다.

## 개발 환경 문제 발생

IntelliJ를 띄워보니 git을 찾지 못한다는 오류 알림이 화면 우하단에 뜬다.

Fix를 눌러 설정화면으로 가보면 아래와 같은 화면이 나오는데 Test를 클릭해보면 `xcrun`이 어쩌고 `developer path`가 어쩌고 하는 오류가 난다.(캡처를 못했..)

![Imgur](https://i.imgur.com/IZnGDoP.png)

## 해결

해결 방법은 *한 마디로 xcode commandline tools를 재설치* 하면 된다.

>xcode-select --install

를 실행하고 나오는 팝업창에서 '설치'를 클릭해서 설치하면 된다.

## 확인

터미널에서 `make --version`을 실행하면 다음과 같이 정상적으로 나온다.

```
🍺  make --version
GNU Make 3.81
Copyright (C) 2006  Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.

This program built for i386-apple-darwin11.3.0
```

IntelliJ 에서 git 설정 테스트해보면 정상적으로 잘 나온다.

![Imgur](https://i.imgur.com/tovne5h.png)
