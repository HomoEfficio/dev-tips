# Mock 테스트의 헛점

Mock 테스트에는 겪어보고 생각해보면 당연하기도 한데 사실은 실패인데도 테스트는 통과로 나오는 False Positive 위험이 있다. 아마 False Negative도 있을테고.

그렇다고 자주 발생하는 것은 아니니 까먹기 전에 적어놔보자.

## 상황

스프링 부트 1.X 에 포함된 spring-data-commons-1.X 의 `CrudRepository` 인터페이스에 선언돼 있는 메서드 중에서 **ID를 통한 검색이 포함되는 `findOne(ID)`, `exists(ID)`, `delete(ID)` 메서드는 ID가 null 이면 IllegalArgumentException을 던지게 돼 있다.**

이제 다음과 같이 흔한 저장 코드를 한 번 보자.

```java
@Override
public MemberResp save(MemberReq memberReq) {
    return MemberResp.from(
            memberRepository.findById(memberReq.getId())  // <-- 여기!!
                    .map(memberReq::updateEntity)
                    .orElseGet(() -> memberRepository.save(memberReq.mapToEntity())));
}
```

위에 `memberRepository.findById(ID)`가 있는데, 1.X 의 `CrudRepository`에는 `findById(ID)`가 없었다. 그래서 `findById(ID)` 메서드를 사용하려면 `MemberRepository`에 직접 선언해줘야 한다.

그리고 **스프링이 `MemberRepository` 인터페이스를 통해 실제 만들어 내는 `findById(ID)`의 구현체는, `findById(ID)`가 포함돼있지 않은 `CrudRepository`와 관계가 없으며 ID가 null 이어도 동작을 한다.** 그래서 위 코드는 `memberReq.getId()`가 null 이더라도 문제 없이 잘 동작한다.

위 코드는 아래와 같은 테스트 코드로 나름 보호받고 있었다.

```java
private MemberService memberService;

@Mock
private MemberRepository memberRepository;

@BeforeEach
public void setup() {
    MockitoAnnotations.initMocks(this);
    memberService = new MemberServiceImpl(memberRepository);
}

@Test
void create_new_member() {
    MemberReq memberReq = new MemberReq(null, "homo.efficio@gmail.com", "Abcd!234", "돌+아이");
    given(memberRepository.findById(null))  // <-- 여기!!
            .willReturn(Optional.empty());
    given(memberRepository.save(argThat(member -> member.getEmail().equals("homo.efficio@gmail.com"))))
            .willReturn(memberReq.mapToEntity());


    MemberResp memberResp = memberService.save(memberReq);


    verify(memberRepository).findById(null);
    verify(memberRepository).save(any());
    assertThat(memberResp.getEmail()).isEqualTo("homo.efficio@gmail.com");
    assertThat(memberResp.getNickname()).isEqualTo("돌+아이");
}
```

## 변화

위 Mock을 사용한 테스트 코드를 그대로 스프링 부트 2.X 에서 얹어서 실행해보면 자연스럽게 통과한다. 그런데 DataJpaTest나 통합테스트 또는 실제 운영 환경 등 Mock이 아닌 실제 Repository가 호출되는 상황에서는 IllegalArgumentException 이 발생한다. 즉, **실제로는 실패인데 Mock 테스트에서는 통과로 나오는 False Positive가 발생**한 것이다.

이유는 다음과 같다.

- 스프링 부트 2.X 에서 사용되는 spring-data-commons-2.X 에서는 `CrudRepository`에 있는 `findOne(ID)` 메서드가 `findById(ID)`로 대체되면서, 
- `findById(null)`은 IllegalArgumentException을 던지도록 변경됐는데, 
- Mock Repository의 `findById(ID)`는 `CrudRepository`에 구애받지 않고 null인 ID도 예외 없이 받아들이기 때문이다.

## 정리

이처럼 **Mock 객체는 실제 사용되는 코드에 걸려있는 제약 사항에 구애 받지 않을 수 있으므로, False Positive를 발생시킬 위험이 있다.**

일찍 드러나지 않을 수도 있는 위험이므로 Mock 객체를 통해 호출되는 메서드는 제약 사항도 함께 고려하는 습관을 들이는 것이 좋겠다.
