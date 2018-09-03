# 스프링부트 static 파일 위치

다음과 같이 `src/main/resources/static/css` 디렉터리 아래에 두고,

![Imgur](https://i.imgur.com/d97HQ4x.png)

FreeMarker 등 템플릿 파일에서는 다음과 같이 참조하면 된다.

![Imgur](https://i.imgur.com/UIpI6O7.png)

인텔리제이에서는 위와 같이 쓰면 노란 물결 무늬 경고가 나오고 `/static/css/custom.css`라고 써야 물결 무늬 경고가 사라지지만, 무시하고 위와 같이 `/css/custom.css`로 작성해야 제대로 css 파일을 읽어온다.

결국 `src/main/resources/static` 디렉터리가 템플릿 파일에서는 `/`로 인식된다.
