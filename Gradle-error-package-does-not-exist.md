# Gradle error: package does not exist

Gradle로 멀티 모듈 프로젝트를 구성해서 스프링부트 애플리케이션끼리 compile 의존 관계를 만들어 사용하다보면,  
물리적으로 분명히 존재하고 CLI에서 `./gradlew compileJava`로는 에러 없이 잘 수행되는데도 불구하고,
IDE에서 컴파일을 포함하는 Gradle task를 수행하면 다음과 같은 에러가 날 때가 있다.

예를 들어 다음 그림과 같이,
`quartz-job` 모듈이 `quartz-scheduler` 모듈에 의존하고,  
`quartz-scheduler` 모듈 안에는 `io.homo_efficio.quartz.scheduler.entity` 패키지가 분명히 존재하는데도,  
`error: io.homo_efficio.quartz.scheduler.entity package does not exist' 라는 에러가 난다.

![Imgur](https://i.imgur.com/3ja2VWw.png)

이유는 딱부러지게 설명할 정도로 이해하진 못 했는데, Gradle SpringBoot 플러그인과 관계가 있는 것 같고, `bootJar`와 관련이 있는 것 같다.

여튼 해결 방법은 다음과 같이 참조되는 모듈 쪽(`quartz-scheduler`)의 `build.gradle`에 다음 내용을 추가하고,  
의존하는 모듈 쪽(`schedule-job`)의 빌드 태스크를 실행하면 에러 없이 잘 빌드 된다.

![Imgur](https://i.imgur.com/aSIRZ0q.png)
