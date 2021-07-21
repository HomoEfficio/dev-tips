# Spring messages file share across multiple module

모듈을 잘게 쪼개서 개발/운영할 때 장점이 많아서 멀티 모듈 프로젝트를 구성할 때가 많다.

그런데 이렇게 하다보면 설정이나 구성 등에서 중복이 많이 발생하기 마련이다.

예외나 알림 등 메시지 문자열을 담고 있는 messages 파일도 예외가 아닌데, 똑같은 파일을 모듈 마다 복사해서 개발을 진행하자니 생산성 저하가 발생한다.

어떻게 하면 한 군데 모아놓고 여러 모듈에서 읽어서 사용할 수 있을까?

아쉽지만 스프링 차원에서는 해결 방법이 없는 걸로 보인다. 어찌보면 당연하다. 라이브러리 모듈을 제외하면 모듈이 곧 애플리케이션이기도 한데 다른 애플리케이션의 자원에 접근한다는 것 자체가 허용되면 안 되는 일이기도 하다.

아무래도 답은 프로젝트 구성 바탕인 Gradle 에서 찾아야 할 것 같다. 이렇게 생각해보니 어렵게 생각할 것 없이 걍 복사 task 를 하나 추가하면 된다.

메시지 파일을 `:common:lib` 모듈에 모아놨다고 하면, 개별 모듈의 build.gradle 파일에 아래와 같은 복사 태스크를 추가하면 된다.

```kotlin
tasks.register<Copy>("myCopyMessages") {
    description = "message 파일을 공동 활용할 수 있도록 common 에 두고 개별 모듈 빌드 시 복사"
    from(project(":common:lib").projectDir.resolve("src/main/resources/message"))
    into("$buildDir/resources/main/message")
}
tasks.processResources {
    dependsOn("myCopyMessages")
}
```

