# Gradle 빌드 오류

gradle 4.2 환경이었는데 빌드 시 다음과 같은 오류가 발생했다.

```
FAILURE: Build failed with an exception.

* What went wrong:
Could not resolve all dependencies for configuration ':dashboard-api:detachedConfiguration1'.
> Could not determine artifacts for javax.ws.rs:javax.ws.rs-api:2.1
   > Could not get resource 'http://NEXUS-REPO-HOST/repository/maven-public/javax/ws/rs/javax.ws.rs-api/2.1/javax.ws.rs-api-2.1.$%7Bpackaging.type%7D'.
      > Could not HEAD 'http://NEXUS-REPO-HOST/repository/maven-public/javax/ws/rs/javax.ws.rs-api/2.1/javax.ws.rs-api-2.1.$%7Bpackaging.type%7D'. Received status code 400 from server: Invalid repository path
```

`gradle clean`도 해보고 Jenkins에서 Workspace를 지워도 봤지만 계속 동일한 에러 발생

한참 검색해도 못 찾아서 https://github.com/HomoEfficio/dev-tips/blob/master/Gradle%20Wrapper%20버전%20변경.md 을 참고해서 gradle wrapper의 버전을 4.10.2 로 업그레이드하니 위 문제가 해결되었다.

다시 검색해보니 https://github.com/gradle/gradle/issues/3065#issuecomment-364086504 여기에도 버전을 올려서 해결했다고 나온다.

---

인텔리제이 2020.1, Gradle 4.5.1, Java 8 에서 gradle 프로젝트를 임포트하면 의존 라이브러리들을 다운로드 하다가 끝까지 받지 못 하고 중간에 멈추면서 결국 build error 가 나는 현상이 있다.

어처구니 없지만 복불복인 것 같다.

`.idea/`, `.gradle/` 두 디렉터리 지우고 다시 임포트, 지우고 다시 임프토 몇 번 하다보면 된다. 운 좋으면 몇 번이면 될테고 운 나쁘면 한 시간..

