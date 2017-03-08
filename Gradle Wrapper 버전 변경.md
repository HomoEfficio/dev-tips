# Gradle Wrapper 버전 변경

프로젝트 루트(gradlew 파일이 있는) 디렉토리에서 다음과 같은 명령을 실행하면 된다.

>./gradlew wrapper --gradle-version 3.4.1

명령이 설공적으로 실행되면 `gradle/wrapper/gradle-wrapper.jar`와 `gradle/wrapper/gradle-wrapper.properties` 파일의 `distributionUrl`의 버전이 위 명령에서 지정한 버전으로 변경된다.

- gradle/wrapper/gradle-wrapper.properties
    
    ```
    #Wed Mar 08 16:37:38 KST 2017
    distributionBase=GRADLE_USER_HOME
    distributionPath=wrapper/dists
    zipStoreBase=GRADLE_USER_HOME
    zipStorePath=wrapper/dists
    distributionUrl=https\://services.gradle.org/distributions/gradle-3.4.1-bin.zip
    ```
