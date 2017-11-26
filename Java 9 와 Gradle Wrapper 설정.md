# Java 9.0.1과 Gradle Wrapper 설정

Java 9.0.1을 새로 깔고 Gradle Wrapper를 사용하는 프로젝트를 열어보니 의존 라이브러리 import가 안 되고 아래와 같은 에러가 뜬다.

![Imgur](https://i.imgur.com/9SCQOVu.png)

이를 해결하려면 아래와 같이 gradle-wrapper.properties 파일에서 `distributionUrl`에 명시된 gradle의 버전을 4.3.1로 바꿔주면 된다.

![Imgur](https://i.imgur.com/j137306.png)
