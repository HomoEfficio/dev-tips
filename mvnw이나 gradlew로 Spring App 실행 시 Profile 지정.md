# mvnw이나 gradlew로 Spring App 실행 시 Profile 지정

`./mvnw -Dspring.profiles.active=local` 이나 `./gradlew -Dspring.profiles.active=local` 로 실행해도 `local` 프로파일로 실행되지 않고 그냥 `default` 프로파일로 실행되면, 아래와 같이 `SPRING_PROFILES_ACTIVE=local`을 앞에 설정해주고 `mvnw`나 `gradlew`를 실행하면 프로파일이 제대로 먹는다.

```
SPRING_PROFILES_ACTIVE=local ./mvnw
```
또는
```
SPRING_PROFILES_ACTIVE=local ./gradlew clean bootRun
```
