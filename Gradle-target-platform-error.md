# Gradle Target Platform Error

그레이들 프로젝트 초기 셋업 시 다음과 같이 target platform 어쩌구 에러를 만날 때가 있다.

![Imgur](https://i.imgur.com/8ymVnrd.png)

1.8을 사용해서 13 용 target platform 을 만들 수 없다는 얘기다.

13은 다음과 같이 `build.gradle`에 지정돼 있다.

```groovy
...
sourceCompatibility = '13'
...
```

다음과 같이 Gradle에서 사용하는 JDK 버전을 맞춰주면 된다.

![Imgur](https://i.imgur.com/X0zpHRu.png)

