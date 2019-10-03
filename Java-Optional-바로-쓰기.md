# Java Optional 바로 쓰기

Brian Goetz는 [스택오버플로우](https://stackoverflow.com/questions/26327957/should-java-8-getters-return-optional-type/26328555#26328555)에서 `Optional`을 만든 의도에 대해 다음과 같이 말했다.

>... it was not to be a general purpose Maybe type, as much as many people would have liked us to do so. Our intention was to provide a limited mechanism for library method return types where there needed to be a clear way to represent "no result" ...
>
>`Optional`은 많은 사람들이 우리(자바 언어 설계자)에게 기대했던 범용적인 `Maybe` 타입과는 다르다. 라이브러리 메서드의 반환값이 '없음'을 명백하게 표현할 필요가 있는 곳에서만 제한적으로 사용할 수 있는 메커니즘을 제공하는 것이 우리가 `Optional`에 담은 의도였다.

뭔 소린지 아리까리하지만 요는 **(이유야 있겠지만) 사람들이 기대하는 것과는 다르게 만들었다는..**  
그럼에도 불구하고 사람들은 기대했던 새로 사용해버려서 [주의사항이 26가지](https://dzone.com/articles/using-optional-correctly-is-not-optional)나 되었..

어쨌든 우리는 만드는 시스템에 해가 없어야 하므로 **`Optional` 사용 시 주의 사항을 자바8 기준으로 갈무리**해봤다.


## isPresent()-get() 대신 orElse()/orElseGet()/orElseThrow()

>**`Optional`은 비싸다. `if (optMember.isPresent()) { ... }`로 하려면 `Optional` 쓰지 말고 `if (member != null) { ... }`를 쓰자.**

```java
// 안 좋음
Optional<Member> member = ...;
if (member.isPresent()) {
    return member.get();
} else {
    return null;
}

// 좋음
Optional<Member> member = ...;
return member.orElse(null);



// 안 좋음
Optional<Member> member = ...;
if (member.isPresent()) {
    return member.get();
} else {
    throw new NoSuchElementException();
}

// 좋음
Optional<Member> member = ...;
return member.orElseThrow(() -> new NoSuchElementException());
```


## orElse(new ...) 대신 orElseGet(() -> new ...)

>**`Optional` 값이 없을 때 새로운 객체를 생성하거나 새로운 연산을 유발한다면 `orElse()`가 아니라 `orElseGet()`을 쓰자.**

`orElse(...)`는 `...`가 새 객체 생성이나 새로운 연산을 유발하지 않고 이미 생성되었거나 계산된 값일 때만 사용한다.  
**`orElse(...)`에서 `...`는 값이 있든 없든 무조건 실행**되고, 값이 없으면 실행된 값이 반환되고, **값이 있으면 실행된 값이 무시**된다.

`orElseGet(Supplier)` 에서 `Supplier`는 값이 없을 때만 실행된다. 따라서 없을 때만 새 객체를 생성하므로 바람직하다.

```java
// 안 좋음
Optional<Member> member = ...;
return member.orElse(new Member());  // member가 있든 없든 new Member()는 무조건 실행됨

// 좋음
Optional<Member> member = ...;
return member.orElseGet(Member::new);  // member가 없을 때만 new Member()가 실행됨

// 좋음
Member EMPTY_MEMBER = new Member();
...
Optional<Member> member = ...;
return member.orElse(EMPTY_MEMBER);  // 이미 생성됐거나 계산된 값은 사용해도 좋음
```


## 단지 값을 얻을 목적이라면 `Optional` 대신 `null` 비교

>**`Optional`은 비싸다. 따라서 단순히 값 또는 `null`을 얻을 목적이라면 `Optional` 대신 `null` 비교를 쓰자.**

```java
// 안 좋음
return Optional.ofNullable(status).orElse(READY);

// 좋음
return status != null ? status : READY;
```


## `Optional` 대신 비어있는 컬렉션 반환

>**`Optional`은 비싸다. 그리고 컬렉션은 `null`이 아니라 비어있는 컬렉션을 반환하는 것이 좋을 때가 많다. 따라서 컬렉션은 `Optional`로 감싸서 반환하지 말고 비어있는 컬렉션을 반환하자.**

```java
// 안 좋음
List<Member> members = team.getMembers();
return Optional.ofNullable(members);

// 좋음
List<Member> members = team.getMembers();
return members != null ? members : Collections.emptyList();
```

마찬가지 이유로 Spring Data JPA Repository 메서드 선언 시 다음과 같이 컬렉션을 `Optional`로 감싸서 반환하는 것은 좋지 않다. **컬렉션을 반환하는 Spring Data JPA Repository 메서드는 `null`을 반환하지 않고 비어있는 컬렉션을 반환해주므로 `Optional`로 감싸서 반환할 필요가 없다.**

```java
// 안 좋음
public interface MemberRepository<Member, Long> extends JpaRepository {
    Optional<List<Member>> findAllByNameContaining(String part);
}

// 좋음
public interface MemberRepository<Member, Long> extends JpaRepository {
    List<Member> findAllByNameContaining(String part);
}
```

## `Optional`을 필드로 사용 금지

>**`Optional`은 필드에 사용할 목적으로 만들어지지 않았으며, `Serializable`을 구현하지 않았다. 따라서 `Optional`은 필드로 사용하지 말자.**

```java
// 안 좋음
public class Member {

    private Long id;
    private String name;
    private Optional<String> email = Optional.empty();
}

// 좋음
public class Member {

    private Long id;
    private String name;
    private String email;
}
```

## `Optional`을 생성자나 메서드 인자로 사용 금지

>**`Optional`을 생성자나 메서드 인자로 사용하면 이를 호출하는 쪽에서 항상 `Optional`을 만들어서 전달해줘야 하는데, 이는 별다른 효용이 없다. 별다른 효용이 없는 곳에 비싼 `Optional`을 사용하지 말자.**


## `Optional`을 컬렉션의 원소로 사용 금지

>**`Optional`은 한다.**

of(), ofNullable() 혼동
OptionalInt, OptionalLong, OptionalDouble
equality에 get() 불필요
map(), flatMap()

