# JPA 영속 객체 사용 시 주의 사항

## 문제

아래 코드(살짝 슈도코드)의 문제점은?

```java
@Transactional
public void method1() {
    Long memberId = 1L;
    Member member = memberRepository.findById(memberId)
        .orElseThrow(() -> new RuntimeException("회원 [" + memberId + "]은 존재하지 않습니다."));
    ...
    method2(memberId);
    ...
    member.change(newEmail);
    ...
}

private void method2(Long memberId) {
    ...
    OtherObject other = new OtherObject(otherInfo, memberId);
    ...
    method3(other);
}

private void method3(OtherObject other) {
    ...
    other.otherMethod1();
    ...
}

public class OtherObject {
    ...
    @Transactional
    public void otherMethod1() {
        Member member = memberRepository.findById(memberId)
            .orElseThrow(() -> new RuntimeException("회원 [" + memberId + "]은 존재하지 않습니다."));
        member.changeAddress(newAddress);
        ...
    }
    ...
}

```

## 답

OtherObject.otherMethod1()에서 실행한 회원(1L)의 집 주소 변경은 반영되지 않는다.

## 그래서?

>영속 상태의 객체 ID를 여기저기 인자로 전달하면,  
>위와 같은 부작용이 발생할 수 있고 원인을 찾기도 어려우므로,  
>영속 상태 객체는 ID 대신 객체 자체를 전달하는 것이 좋다.
