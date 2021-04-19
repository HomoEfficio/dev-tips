# IntelliJ, JHipster, Gradle 실행 설정

보통 IntelliJ + Gradle + Java/Kotlin으로 개발할 때 Grdale 설정 중 Build and run 항목 2가지 모두 IntelliJ IDEA 로 설정해둔다.

이유는 최적화 덕분에 실행이 더 빠르기 때문이다. 라고 아래 그림에도 써있다.  
그런데 그 바로 아랫줄은 잘 안 읽어봤었는데 IntelliJ IDEA 가 모든 Gradle 플러그인을 지원하지는 않으므로 빌드가 제대로 되지 않는 경우도 있으니 주의하라고 써있다.  
여태 이런 경우를 한 번도 겪지 못 했는데 드디어 겪었다.

JHipster로 구성한 프로젝트를 IntelliJ IDEA로 실행하면 localhost:8080 접근 시 404 에러가 나고 멀쩡히 존재하는 Swagger 페이지도 404가 난다.  
그런데 아래와 같이 Build and run using 항목을 Gradle 로 선택하면 정상적으로 나온다.

# 마무리

IntelliJ + Gradle + Java/Kotlin 로 개발하는데 IntelliJ에서 실행 시 뭔가 제대로 동작하지 않는다면,  
아래와 같이 Build and run using 항목을 Gradle 로 선택하고 실행하자.

![Imgur](https://i.imgur.com/3YJ6oyI.png)

