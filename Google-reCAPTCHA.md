# Google reCAPTCHA

- CAPTCHA: Completely Automated Public Turing test to tell Computers and Humans Apart
  - 봇에 의한 요청인지 사람에 의한 요청인지 구분해서 DoS 같은 스팸 공격을 막을 수 있는 보안 기법
  - 구글 리캡차는 CAPTCHA 구현체 중 하나

## 주요 흐름

### 설정

1. 구글 리캡차 어드민 콘솔(https://www.google.com/recaptcha/about/)에서 리캡차를 이용할 서비스의 도메인 등을 입력하고 등록한다.

![Imgur](https://i.imgur.com/uWmdwOD.png)

2. 등록되면 사이트키와 시크릿키가 발급된다.

![Imgur](https://i.imgur.com/LA9H0ns.png)

3. 리캡차를 적용할 HTML 페이지에 다음과 같이 리캡차 컴포넌트가 표시될 div 영역을 정의한다. 공식 문서는 [여기](https://developers.google.com/recaptcha/docs/v3)

```html
<div class="g-recaptcha mb-2" data-sitekey="발급받은 사이트키"></div>
```

4. 리캡차 컴포넌트 자바스크림트 임포트

```html
<script src="https://www.google.com/recaptcha/api.js"></script>
```

5. 시크릿키를 안전한 곳에 저장(편의상 application.yml에 저장한다고 가정)

### 동작

1. 리캡차가 적용된 화면에 접속하면 사용자가 리캡차 컴포넌트의 체크박스 클릭

2. 그림 퀴즈 등 표시

3. 사용자가 그림 퀴즈 등을 보고 적절한 응답을 선택하고 요청 전송

4. 서버는 application.yml에 저장된 시크릿키를 읽어서 구글 리캡차 API 호출(사용자IP, 시크릿키, 사용자선택응답)


