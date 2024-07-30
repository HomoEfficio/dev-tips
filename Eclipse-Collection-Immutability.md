# Eclipse Collection과 Immutability

자바에서 불변 컬렉션(Immutable Collection)을 만들 때 [Eclipse Collection](https://eclipse.dev/collections/)을 사용하면 편리하다.

Eclipse Collection의 ImmutableCollection 인터페이스는 java.util.Collection 인터페이스를 상속받지 않아서,  
java.util.Collection 인터페이스에 포함돼 있는 `add(), addAll(), remove(), removeAll(), removeIf()` 등과 같은 mutable 메서드가 아예 없다.
그래서 컴파일 타임에 컬렉션의 불변성(Immutability)이 보장된다.

~하지만 Eclipse Collection의 ImmutableMap은 컴파일 타임에는 불변성이 보장되지 않는다.~  
Eclipse Collection의 ImmutableMap은 컴파일 타임에 불변성이 보장되는 것이 있고 보장되지 않는 것이 있다.

ImmutableUnifiedMap을 사용하면 ImmutableUnifiedMap이 상속받고 있는 AbstractImmutableMap 클래스가 immutable 메서드를 포함하고 있는 java.util.Map 인터페이스를 구현하고 있어서 컴파일 타임에 mutable 메서드를 호출할 수 있어서 컴파일 타임 불변성은 보장되지 않는 대신, java.util.Map과 호환되므로 java.util.Map 타입 변수에 할당할 수 있다. java.util.Map과 호환되어 mutable 메서드를 호출할 수는 있지만, 호출하면 모두 UnsupportedOperationException를 던지게 돼 있어서 런타임 불변성은 보장된다.

`Maps.immutable.ofAll(mutableMap)`을 사용해서 ImmutableMap 을 만들면 AbstractImmutableMap 클래스를 사용하지 않아 java.util.Map 인터페이스를 구현하고 있지 않으므로 컴파일 타임에 mutable 메서드를 호출할 수 없어서 컴파일 타임 불변성이 보장되며,
이렇게 만들어진 ImmutableMap 은 java.util.Map 인터페이스를 구현하고 있지 않으므로 당연히 java.util.Map에 할당할 수 없다.

다만 아래 사례 중 마지막 사례를 보면, ImmutableCollection이나 ImmutableMap이 중첩된 컬렉션이나 맵을 포함하는 경우, 포함된 컬렉션이나 맵의 실질 타입에 따라 내부 불변성은 유지되지 않을 수도 있음에 유의해야 한다. 내부 불변성까지 완전하게 유지하려면 포함되는 컬렉션이나 맵도 불변이어야 한다.

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

// 가변 해시맵으로부터 ImmutableUnifiedMap 생성
ImmutableMap<String, String> immutableUnifiedMap = new ImmutableUnifiedMap<>(mutableHashMap);
immutableUnifiedMap.forEach((k, v) -> System.out.println(k + ": " + v));
immutableUnifiedMap.remove("k1");  // UnsupportedOperationException
// java.util.Map 타입 변수에 할당될 수 있고, java.util.Map을 반환하는 메서드의 반환값으로 사용할 수 있다.
Map<String, String> aMap = immutableUnifiedMap;

// 불변 맵을 가변 맵으로 캐스팅
Map<String, String> castedMap = immutableUnifiedMap.castToMap();
castedMap.remove("k1");  // Map으로 타입변환해도 UnsupportedOperationException 발생하며 런타임 Immutability 보장

// 가변 해시맵으로부터 ImmutableMap 생성
ImmutableMap<String, String> immutableMap = Maps.immutable.of(mutableHashMap);
immutableMap.forEachKeyValue((k, v) -> System.out.println(k + ": " + v));
immutableMap.remove  // 자동완성도 되지 않고 컴파일 에러
Map<String, String> aMap = immutableMap;  // 할당되지 않고 컴파일 에러, java.util.Map을 반환하는 메서드의 반환값으로 사용할 수 없다.


// 중첩된 맵이 불변맵이면 ImmutableMap 에 저장되었다가 get 으로 읽어와도 불변
Map<String, String> innerMap11 = Map.of("m1k1", "m1v1", "m1k2", "m1v2");
ImmutableMap<String, Map<String, String>> immutableMap11 = Maps.immutable.of("k1", innerMap11);
Map<String, String> innerMap12 = immutableMap11.get("k1");  // innerMap11 과 동일한 참조, ImmutableCollections$MapN 타입
innerMap12.put("m12k2", "m12k2");  // UnsupportedOperationException

// 중첩된 맵이 가변맵이면 ImmutableMap 에 저장되었다가 get 으로 읽어오면 원래대로 가변이며 ImmutableMap 도 실질적으로 불변성을 유지하지 못함
Map<String, String> innerMap21 = new HashMap<>();
innerMap21.put("m1k1", "m1v1");
innerMap21.put("m1k2", "m1v2");
ImmutableMap<String, Map<String, String>> immutableMap21 = Maps.immutable.of("k1", innerMap21);
Map<String, String> innerMap22 = immutableMap21.get("k1");  // innerMap21 과 동일한 참조, HashMap 타입
innerMap22.put("m22k2", "m12k2");  // 추가됨

```

반면 구글의 [Guava ImmutableCollection](https://guava.dev/releases/33.2.1-jre/api/docs/com/google/common/collect/ImmutableCollection.html)은 java.util.Collection 인터페이스를 상속받고 있어서 컴파일 타임 불변성이 보장되지 않는다.

아쉽지만 불변 객체를 만드는 데 유용한 [immutables](https://github.com/immutables/immutables) 라이브러리도 내부적으로 Guava Collection을 사용하고 있어서 컴파일 타임 불변성은 보장되지 않는다.

