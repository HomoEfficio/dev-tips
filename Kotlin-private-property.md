# Kotlin private property 사용 주의

아래 객체를 JSON으로 만들면 어떻게 나올까?

```kotlin
class Abc(
    private val id: String,
    private val name: String
)

val myAbc = Abc("homo.efficio", "효율의 인간")
```

이렇게 나온다.
```json
{}
```

이유는 Kotlin에서 getter는 늘 public으로 만들어지는 게 아니라 기본적으로 해당 프로퍼티의 접근 제한자와 동일한 접근 제한자가 지정되기 때문이다.

>Classes, objects, interfaces, constructors, functions, properties and their setters can have visibility modifiers. **Getters always have the same visibility as the property.**
>
>https://kotlinlang.org/docs/visibility-modifiers.html#classes-and-interfaces

그래서 private으로 선언한 id, name property는 private getter가 생성되어 외부에서 읽을 수도 없고,  
그래서 JSON 직렬화 시 위와 같이 비어있는 `{}`가 만들어진다.

그런데 실제로 IDE를 이용해서 Decompile 후 Java 파일로 보면 아래와 같이 아예 getter가 만들어지지도 않는다.

```java
@Metadata(
   ...
)
public final class Abc {
   private final String id;
   private final String name;

   public Abc(@NotNull String id, @NotNull String name) {
      Intrinsics.checkNotNullParameter(id, "id");
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.id = id;
      this.name = name;
   }
}
```

암튼 외부에서 읽을 수도 없기는 마찬가지다.

그러니 ReadOnly 이면서 외부에서 읽을 수 있어야 하는 property는 private을 빼고 그냥 아래와 같이 선언해야 한다.

```kotlin
class Abc(
    val id: String,
    val name: String
)
```

어차피 ReadOnly라서 setter가 만들어지지도 않으므로 private을 빼도 외부에서 값이 변경될 일은 없다.

참고로 위와 같이 private 을 사용하지 않을 때 Java 파일로 보면 다음과 같이 필드 자체는 private로 선언되고 public getter가 생성되므로, 애초에 멋 모르고 private을 붙였을 때 의도한 것과 같은 결과가 나온다. 따라서 Kotlin에서는 의미 없이 습관적으로 private을 붙일 필요가 없고, 그래서도 안 된다.

```java
@Metadata(
   ...
)
public final class Abc {
   @NotNull
   private final String id;
   @NotNull
   private final String name;

   @NotNull
   public final String getId() {
      return this.id;
   }

   @NotNull
   public final String getName() {
      return this.name;
   }

   public Abc(@NotNull String id, @NotNull String name) {
      Intrinsics.checkNotNullParameter(id, "id");
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.id = id;
      this.name = name;
   }
}
```
