# JUnit4 와 JUnit5 를 함께 쓰기

레거시 코드가 JUnit4 를 사용하고 있는데, 앞으로는 JUnit5 를 쓰고 싶다.

그렇다고 기존 JUnit4 테스트코드를 모두 JUnit5 로 바꾸는 건 일이 너무 크다.

어떻게 하면 그냥 둘을 사이좋게 같이 쓸 수 있을까?

maven 환경이라면 아래 링크와 같이 pom.xml 파일을 설정하고

https://stackoverflow.com/a/47158584

JUnit5 테스트 코드를 사용할 때 `@Test` 애너테이션을 다음과 같이 jupiter로 지정해서 붙이면 된다.

파일 전체를 JUnit5 로 테스트 한다면 아래와 같이 import 문에서만 jupiter 로 해주면 되고,

>import org.junit.jupiter.api.Test;

같은 파일 안에서 일부 메서드만 JUnit5 로 한다면 아래와 같이 패키지명을 포함해서 애노테이션을 지정해준다. (아직 미확인)

>@org.junit.jupiter.api.Test


