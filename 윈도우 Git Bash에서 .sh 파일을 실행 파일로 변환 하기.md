# 윈도우 Git Bash에서 .sh 파일을 실행 파일로 변환하기


# 문제

윈도우 Git Bash에서 다음과 같이 .sh 파일에 `chmod` 명령을 사용해도 실행 파일로 변환되지 않는다.

![Imgur](http://i.imgur.com/rEjJJnN.png)

어떻게 해야 실행 시킬 수 있을까?

## 답

>sh 파일의 첫 행이 `#!`로 시작하면 `chmod`를 하지 않고도 Git Bash가 알아서 실행 파일로 인식한다.

### 원래 파일 내용

![Imgur](http://i.imgur.com/MkwARo0.png)

### `#!/bin/sh` 추가

![Imgur](http://i.imgur.com/ARorfuU.png)

사실 뒤에 `/bin/sh`를 안 붙이고 그냥 `#!`만 추가해도 실행 파일로 인식되며 실행도 잘 되기는 한다. 하지만 괜히 엉뚱한 짓 하지 말고 그냥 붙여주자.

### 실행 파일 인식 확인

![Imgur](http://i.imgur.com/ZXgKlTI.png)

### 실제 실행

![Imgur](http://i.imgur.com/GrM14u2.png)

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
