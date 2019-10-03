# Java Optional 바로 쓰기

Brian Goetz는 [스택오버플로우](https://stackoverflow.com/questions/26327957/should-java-8-getters-return-optional-type/26328555#26328555)에서 `Optional`을 만든 의도에 대해 다음과 같이 말했다.

>... it was not to be a general purpose Maybe type, as much as many people would have liked us to do so. Our intention was to provide a limited mechanism for library method return types where there needed to be a clear way to represent "no result" ...
>
>`Optional`은 많은 사람들이 우리(자바 언어 설계자)에게 기대했던 범용적인 `Maybe` 타입과는 다르다. **라이브러리 메서드의 반환값이 '없음'을 명백하게 표현할 필요가 있는 곳에서만 제한적으로 사용할 수 있는 메커니즘을 제공하는 것이 `Optional`을 만든 의도**였다.

뭔 소린지 아리까리하지만 요는 **반환값이 '없음'을 나타내는 것이 주목적**이며, (이유야 있겠지만) **사람들이 기대하는 것과는 다르게 만들었다는..**  
그럼에도 불구하고 사람들은 기대했던 대로 사용해버려서 [주의사항이 26가지](https://dzone.com/articles/using-optional-correctly-is-not-optional)나 되었.. (의도와 다른 방식으로 사용되는 것을 허용한 이유는 또 뭘까..)

어쨌든 원래 의도에 맞게 쓰는 것이 가급적 해가 없을 것이고, 우리는 우리가 만드는 시스템에 해를 끼치지 말아야 한다. 그래서 **`Optional` 사용 시 무심결에 잘못 사용하는 안티패턴과 올바른 사용법을 자바8 기준으로 갈무리**해봤다.


## 1. `isPresent()-get()` 대신 `orElse()/orElseGet()/orElseThrow()`

>**이왕에 비싼 `Optional` 쓰기로 한 거 코드라도 줄이자. 설명보다 그냥 코드를 보는 게 훨씬 낫다.**

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


## 2. `orElse(new ...)` 대신 `orElseGet(() -> new ...)`

>**`Optional`에 값이 없을 때만 새로운 객체를 생성하거나 새로운 연산이 수행되도록 `orElse()` 대신 `orElseGet()`을 쓰자.**

**`orElse(...)`에서 `...`는 `Optional`에 값이 있든 없든 무조건 실행**되고, 값이 없으면 실행된 값이 반환되고, **값이 있으면 실행된 값이 무시되고 버려진다.**  
따라서 `orElse(...)`는 `...`가 새 객체 생성이나 새로운 연산을 유발하지 않고 이미 생성되었거나 계산된 값일 때만 사용한다.

**`orElseGet(Supplier)`에서 `Supplier`는 `Optional`에 값이 없을 때만 실행된다. 따라서 `Optional`에 값이 없을 때만 새 객체를 생성하므로 바람직하다.**

```java
// 안 좋음
Optional<Member> member = ...;
return member.orElse(new Member());  // member에 값이 있든 없든 new Member()는 무조건 실행됨

// 좋음
Optional<Member> member = ...;
return member.orElseGet(Member::new);  // member에 값이 없을 때만 new Member()가 실행됨

// 좋음
Member EMPTY_MEMBER = new Member();
...
Optional<Member> member = ...;
return member.orElse(EMPTY_MEMBER);  // 이미 생성됐거나 계산된 값은 orElse()를 사용해도 무방
```

참고로 `orElse(Collections.emptyList())`는 `Collections.emptyList()`가 호출될 때마다 비어있는 리스트를 반환하는 것이 아니라 이미 생성된 static 변수인 `EMPTY_LIST`를 반환하므로 괜찮다. 하지만, 이런 용법은 많이 사용되면 `orElse(new ...)` 같은 안티 패턴을 정상적인 사용법으로 보이게 하는 좋지 않은 착시 효과가 발생할 수 있으므로 **`orElseGet(Collections::emptyList)`를 사용하는 것이 더 좋다.**


## 3. 단지 값을 얻을 목적이라면 `Optional` 대신 `null` 비교

>**`Optional`은 비싸다. 따라서 단순히 값 또는 `null`을 얻을 목적이라면 `Optional` 대신 `null` 비교를 쓰자.**

```java
// 안 좋음
return Optional.ofNullable(status).orElse(READY);

// 좋음
return status != null ? status : READY;
```


## 4. `Optional` 대신 비어있는 컬렉션 반환

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
    List<Member> findAllByNameContaining(String part);  // null이 반환되지 않으므로 Optional 불필요
}
```

## 5. `Optional`을 필드로 사용 금지

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

## 6. `Optional`을 생성자나 메서드 인자로 사용 금지

>**`Optional`을 생성자나 메서드 인자로 사용하면, 호출할 때마다 `Optional`을 생성해서 인자로 전달해줘야 한다. 하지만 호출되는 쪽, 즉 api나 라이브러리 메서드에서는 인자가 `Optional`이든 아니든 `null` 체크를 하는 것이 언제나 안전하다. 따라서 굳이 비싼 `Optional`을 인자로 사용하지 말고 호출되는 쪽에 `null` 체크 책임을 남겨두는 것이 좋다.**

```java
// 안 좋음
public class HRManager {
    
    public void increaseSalary(Optional<Member> member) {
        member.ifPresent(member -> member.increaseSalary(10));
    }
}
hrManager.increaseSalary(Optional.ofNullable(member));

// 좋음
public class HRManager {
    
    public void increaseSalary(Member member) {
        if (member != null) {
            member.increaseSalary(10);
        }
    }
}
hrManager.increaseSalary(member);
```


## 7. `Optional`을 컬렉션의 원소로 사용 금지

>**컬렉션에는 많은 원소가 들어갈 수 있다. 따라서 비싼 `Optional`을 원소로 사용하지 말고 원소를 꺼낼 때나 사용할 때 `null` 체크하는 것이 좋다. 특히 Map은 `getOrDefault()`, `putIfAbsent()`, `computeIfAbsent()`, `computeIfPresent()` 처럼 `null` 체크가 포함된 메서드를 제공하므로, Map의 원소로 `Optional`을 사용하지 말고 Map이 제공하는 메서드를 활용하는 것이 좋다.**

```java
// 안 좋음
Map<String, Optional<String>> sports = new HashMap<>();
sports.put("100", Optional.of("BasketBall"));
sports.put("101", Optional.ofNullable(someOtherSports));
String basketBall = sports.get("100").orElse("BasketBall");
String unknown = sports.get("101").orElse("");

// 좋음
Map<String, String> sports = new HashMap<>();
sports.put("100", "BasketBall");
sports.put("101", null);
String basketBall = sports.getOrDefault("100", "BasketBall");
String unknown = sports.computeIfAbsent("101", k -> "");
```


## 8. `of()`, `ofNullable()` 혼동 주의

>**`of(X)`은 `X`가 `null`이 아님이 확실할 때만 사용해야 하며, `X`가 `null`이면 NullPointerException 이 발생한다.**  
>**`ofNullable(X)`은 `X`가 `null`일 수도 있을 때만 사용해야 하며, `X`가 `null`이 아님이 확실하면 `of(X)`를 사용해야 한다.**

```java
// 안 좋음
return Optional.of(member.getEmail());  // member의 email이 null이면 NPE 발생

// 좋음
return Optional.ofNullable(member.getEmail());



// 안 좋음
return Optional.ofNullable("READY");

// 좋음
return Optional.of("READY");
```


## 9. `Optional<T>` 대신 `OptionalInt`, `OptionalLong`, `OptionalDouble` 사용

>**`Optional`에 담길 값이 `int`, `long`, `double`이라면 Boxing/Unboxing이 발생하는 `Optional<Integer>`, `Optional<Long>`, `Optional<Double>`을 사용하지 말고, `OptionalInt`, `OptionalLong`, `OptionalDouble`을 사용하자.**

```java
// 안 좋음
Optional<Integer> count = Optional.of(38);  // boxing 발생
for (int i = 0 ; i < count.get() ; i++) { ... }  // unboxing 발생

// 좋음
OptionalInt count = OptionalInt.of(38);  // boxing 발생 안 함
for (int i = 0 ; i < count.getAsInt() ; i++) { ... }  // unboxing 발생 안 함
```
