# Test Double

단위 테스트는 테스트 대상 객체의 동작만을 고립 시켜서 테스트하므로, 테스트 대상 클래스가 사용하는 다른 요소는 가짜를 만들어 대신 사용한다.

이런 가짜를 통칭해서 테스트 더블(Test Double)이라고 한다.

Meszaross의 책에서는 5 가지 테스트 더블이 나온다.

- Dummy
- Fake
- Stub
- Spy
- Mock

이 중 Dummy는 테스트 분야에서는 이젠 거의 사용되지 않는 용어이므로 그냥 무시하자.

## Fake

Fake는 설명보다는 그냥 예를 들면 바로 이해가 된다. 테스트에서 실제 DB 대신 사용되는 인메모리 DB가 Fake에 해당한다. 진짜 DB와 마찬가지로 실제로 의미있는 CRUD 동작을 하지만 실제 운영 상에서 사용하는 DB를 대체한다.

## Stub

Stub은 주로 하드 코딩을 통해 특정 값을 반환하도록 만들어진 가짜 구현체다.

```java
public interface MemberService {
  String getMemberName(Long id);
}

public class MemberServiceStub implements MemberService {
  
  @Override
  public String getMemberName(Long id) {
    return "homo.efficio";  // 테스트 목적에 맞는 특정 값을 반환하도록 만든 가짜 구현체
  }
}

public class FriendRecommendationServiceTest {
  
  @Test
  public void id_3_회원을_조회하면_이름은_homo_efficio() throws Exception {

    MemberService memberService = new MemberServiceStub();
    FriendRecommendationService frService = new FriendRecommendationServiceImpl(memberService);

    // frService.getFriendNames() 로직 내부적으로 
    List<String> names = frService.getFriendNames(1L);

    assertThat(names).containsOnly("homo.efficio");
  }
}
```

