# NodeJS, npm install 시 가끔 만나는 node-gyp rebuild 에러

## 기

`npm install`로 라이브러리를 설치하다 보면 중간에 `node-gyp rebuild`를 실행하다가 다음과 같은 에러를 뿜으며 장렬히 전사하는 경우가 있다.

![Imgur](https://i.imgur.com/suuBJe9.png)

에러 메시지로 보자면 protected 함수에 접근할 수 없는 영역에서 protected 함수를 호출하고 있는 걸로 보인다.

## 승

그래서 에러 메시지에 있는대로 820행에 가보니 아래와 같이 `CreateHandle` 함수가 protected 영역에 선언되어 있다.

![Imgur](https://i.imgur.com/MoTGsAV.png)

## 전

편법이긴 하지만 아래와 같이 `CreateHandle` 함수 선언부를 `public` 영역으로 옮겨버리면 된다.

![Imgur](https://i.imgur.com/MjqA7Yh.png)

## 결

다시 `npm install`을 실행하면 `node-gyp rebuild`는 문제 없이 실행된다.

에러 해결 후 뭔가 개운치 않으면 다시 protected 영역으로 원위치 시켜 놓으면 된다.
