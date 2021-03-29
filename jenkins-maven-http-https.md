# Jenkins 에서 http://repo.maven.apache.org/maven2 사용 이슈

Jenkins 빌드에서 대략 다음과 같이 HTTPS 관련 에러가 나면,

```
Downloading: http://repo.maven.apache.org/maven2/org/junit/junit-bom/5.7.1/junit-bom-5.7.1.pom

[ERROR] The build could not read 1 project -> [Help 1]
org.apache.maven.project.ProjectBuildingException: Some problems were encountered while processing the POMs:
[WARNING] 'dependencies.dependency.(groupId:artifactId:type:classifier)' must be unique: com.auth0:java-jwt:jar -> version 3.8.1 vs 3.10.0 @ line 473, column 15
[ERROR] Non-resolvable import POM: Could not transfer artifact org.junit:junit-bom:pom:5.7.1 from/to central (http://repo.maven.apache.org/maven2): Failed to transfer file: http://repo.maven.apache.org/maven2/org/junit/junit-bom/5.7.1/junit-bom-5.7.1.pom. Return code is: 501 , ReasonPhrase:HTTPS Required. @ org.springframework.boot:spring-boot-dependencies:2.1.1.RELEASE, /home1/irteam/.m2/repository/org/springframework/boot/spring-boot-dependencies/2.1.1.RELEASE/spring-boot-dependencies-2.1.1.RELEASE.pom, line 2400, column 25
```

먼저 눈여겨 봐야할 곳은 첫 줄이다.

`Downloading: http://repo.maven.apache.org/maven2/...`

maven central repo 가 2020-01-25 부터는 https 로 전환됐다. 참고: https://stackoverflow.com/questions/59764749/requests-to-http-repo1-maven-org-maven2-return-a-501-https-required-status-an

따라서 http 가 아니라 https 를 사용해야 한다.

그런데 pom.xml 에서 repository 를 분명하게 `https://repo.maven.apache.org/maven2/`로 지정해줘도,  
Jenkins 는 나몰라라 계속 `http://repo.maven.apache.org/maven2/`를 호출한다.

**Jenkins 에 설정된 maven 버전이 3.2.2 였는데, 3.6.3 으로 업그레이드 하니까 pom.xml 에 지정한대로 https 리포지토리를 호출해서 문제가 해결**됐다.



