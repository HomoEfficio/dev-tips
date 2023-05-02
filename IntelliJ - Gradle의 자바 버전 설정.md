# IntelliJ - Gradle의 자바 버전 설정

여러 버전의 자바를 사용하다보면 Gradle 빌드 시 다음과 같은 에러를 마주할 때가 있다.

![Imgur](https://i.imgur.com/lyPS180.png)

자바8를 사용해서 자바9 결과물을 만들 수 없다는 얘기다.

## 원인

다음과 같이 프로젝트 설정에 소스 호환성을 9로 해두고,

![Imgur](https://i.imgur.com/3ONg5Uq.png)

build.gradle 에도 9로 해뒀는데,

![Imgur](https://i.imgur.com/qfZbeMx.png)

다음과 같이 Gradle 설정에는 Java 1.8을 사용하게 돼있기 때문에 발생하는 오류다.

![Imgur](https://i.imgur.com/OGZsFES.png)

## 해결

따라서 다음과 같이 Gradle의 자바 버전을 9로 맞춰주면,

![Imgur](https://i.imgur.com/jsdafmy.png)

정상적으로 빌드 된다.

![Imgur](https://i.imgur.com/JeSV7tk.png)
