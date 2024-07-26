# Eclipse Collection과 Immutability

자바에서 불변 컬렉션(Immutable Collection)을 만들 때 [Eclipse Collection](https://eclipse.dev/collections/)을 사용하면 편리하다.

Eclipse Collection의 ImmutableCollection 인터페이스는 java.util.Collection 인터페이스를 상속받지 않아서,  
java.util.Collection 인터페이스에 포함돼 있는 `add(), addAll(), remove(), removeAll(), removeIf()` 등과 같은 mutable 메서드가 아예 없다.
그래서 컴파일 타임에 컬렉션의 불변성(Immutability)이 보장된다.

하지만 Eclipse Collection의 ImmutableMap은 컴파일 타임에는 불변성이 보장되지 않는다.
이유는 ImmutableMap를 실제로 생성할 때 사용하는 AbstractImmutableMap 클래스가 immutable 메서드를 포함하고 있는 java.util.Map 인터페이스를 구현하고 있기 때문이다.
하지만 mutable 메서드의 구현체는 모두 UnsupportedOperationException를 던지게 돼 있어서 런타임에는 불변성이 보장된다.

홈페이지에 있는 것 외에 아주 간단한 사용 사례를 남겨본다.

```java
// 가변 리스트
List<String> mutableList = new ArrayList<>();
mutableList.add("Mutable");

// 가변 리스트로부터 불변 리스트 생성
ImmutableList<String> immutableList = Lists.immutable.ofAll(mutableList);
ImmutableList.add  // 자동완성도 되지 않고 컴파일 에러

// 불변 리스트를 List 로 캐스팅
List<String> castedList = immutableList.castToList();

// List로 타입이 변환되어 add 를 호출할 수는 있지만 UnsupportedOperationException이 발생하며 런타임 Immutability 보장
castedList.add("Added thru castedList");  


// 가변 해시맵
Map<String, String> mutableHashMap = new HashMap<>();
for (int i = 0; i < 10; i++) {
    mutableHashMap.put("k" + i, "v" + i);
}

// 가변 해시맵으로부터 불변 맵 생성
ImmutableUnifiedMap<String, String> immutableMap = new ImmutableUnifiedMap<>(mutableHashMap);
immutableMap.forEach((k, v) -> System.out.println(k + ": " + v));
immutableMap.remove("k1");  // UnsupportedOperationException

// 불변 맵을 가변 맵으로 캐스팅
Map<String, String> castedMap = immutableMap.castToMap();
castedMap.remove("k1");  // Map으로 타입변환해도 UnsupportedOperationException 발생하며 런타임 Immutability 보장
```

반면 구글의 [Guava ImmutableCollection](https://guava.dev/releases/33.2.1-jre/api/docs/com/google/common/collect/ImmutableCollection.html)은 java.util.Collection 인터페이스를 상속받고 있어서 컴파일 타임 불변성이 보장되지 않는다.

아쉽지만 불변 객체를 만드는 데 유용한 [immutables](https://github.com/immutables/immutables) 라이브러리도 내부적으로 Guava Collection을 사용하고 있어서 컴파일 타임 불변성은 보장되지 않는다.

