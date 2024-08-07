# Eclipse Collection과 Immutability

자바에서 불변 컬렉션(Immutable Collection)을 만들 때 [Eclipse Collection](https://eclipse.dev/collections/)을 사용하면 편리하다.

## Compile-time Immutability and Run-time Immutability

Eclipse Collection의 ImmutableCollection 인터페이스는 java.util.Collection 인터페이스를 상속받지 않아서,  
java.util.Collection 인터페이스에 포함돼 있는 `add(), addAll(), remove(), removeAll(), removeIf()` 등과 같은 mutable 메서드가 아예 없다.
그래서 컴파일 타임에 컬렉션의 불변성(Immutability)이 보장된다.

~하지만 Eclipse Collection의 ImmutableMap은 컴파일 타임에는 불변성이 보장되지 않는다.~  
Eclipse Collection의 ImmutableMap은 컴파일 타임에 불변성이 보장되는 것이 있고 보장되지 않는 것이 있다.

ImmutableUnifiedMap을 사용하면 ImmutableUnifiedMap이 상속받고 있는 AbstractImmutableMap 클래스가 immutable 메서드를 포함하고 있는 java.util.Map 인터페이스를 구현하고 있어서 컴파일 타임에 mutable 메서드를 호출할 수 있어서 컴파일 타임 불변성은 보장되지 않는 대신, java.util.Map과 호환되므로 java.util.Map 타입 변수에 할당할 수 있다. java.util.Map과 호환되어 mutable 메서드를 호출할 수는 있지만, 호출하면 모두 UnsupportedOperationException를 던지게 돼 있어서 런타임 불변성은 보장된다.

`Maps.immutable.ofAll(mutableMap)`을 사용해서 ImmutableMap 을 만들면 AbstractImmutableMap 클래스를 사용하지 않아 java.util.Map 인터페이스를 구현하고 있지 않으므로 컴파일 타임에 mutable 메서드를 호출할 수 없어서 컴파일 타임 불변성이 보장되며,
이렇게 만들어진 ImmutableMap 은 java.util.Map 인터페이스를 구현하고 있지 않으므로 당연히 java.util.Map에 할당할 수 없다.

다만 아래 사례 중 마지막 사례를 보면, ImmutableCollection이나 ImmutableMap이 중첩된 컬렉션이나 맵, primitive가 아닌 객체를 포함하는 경우, 포함된 컬렉션이나 맵, 객체에 따라 내부 불변성은 유지되지 않을 수도 있음에 유의해야 한다.  
**내부 불변성까지 완전하게 보장하려면 포함되는 컬렉션이나 맵, 객체가 불변이어야 한다.**

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

## 성능

`Lists.immutable.ofAll(mutableList)`나 `Maps.immutable.ofAll(mutableMap)`와 같이 **ImmutableCollection이나 ImmutableMap으로 만드는 과정은 일반적인 복사에 비해 성능에서 굉장히 불리할 수 있다**는 점에 유의하자.

아래와 같이 테스트 해보면 리스트 원소가 100만개 일 때는 차이가 무려 500배까지도 난다. ㄷㄷㄷ

이하 모든 코드는 JDK 17을 사용했다.

```java
@Test
void normal_copy_vs_immutable() {
    List<Integer> integers = IntStream.rangeClosed(1, 1_000_000)
            .boxed()
            .collect(Collectors.toList());

    StopWatch sw = new StopWatch("normal vs immutable");

    sw.start("to immutable");
    ImmutableList<Integer> immutableIntegers = Lists.immutable.ofAll(integers);
    System.out.println(immutableIntegers.size());
    sw.stop();

    sw.start("normal copy");
    List<Integer> mutableIntegersCopied = new ArrayList<>(integers);
    System.out.println(mutableIntegersCopied.size());
    sw.stop();
    System.out.println(sw.prettyPrint());
}

=====
StopWatch 'normal vs immutable': running time = 270367791 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
269862250  100%  to immutable
000505541  000%  normal copy
```

흥미로운 것은 리스트의 갯수를 1000만개로 10배 늘리면 normal copy는 수행시간도 대략 10배가 걸리는데, to immutable 은 10%도 늘어나지 않는다.
```
StopWatch 'normal vs immutable': running time = 289116126 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
283223584  098%  to immutable
005892542  002%  normal copy
```

하지만 1억개로 하면 to immutable 도 늘어난다.
```
StopWatch 'normal vs immutable': running time = 583402750 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
426178250  073%  to immutable
157224500  027%  normal copy
```

이렇게 성능까지 살펴보니 이거 써도 되는 건가 싶기도.. =3=3

참고로 kotlin의 toList()를 아래와 같이 추가해보면 결과는 다음과 같다.
100만개 일때는 일반 복사에 비해 kotlin toList()가 7배 정도 느리지만,  
1000만개부터는 큰 차이가 나지 않는다.

```kotlin
sw.start("kotlin toList")
val kotlinList: List<Int> = integers.toList()
println(kotlinList.size)
sw.stop()


100만개
----------------------------------------------------
Seconds       %       Task name
----------------------------------------------------
0.257979875   98%     to immutable
0.000639708   00%     normal copy
0.004159625   02%     kotlin toList

1000만개
----------------------------------------------------
Seconds       %       Task name
----------------------------------------------------
0.278435583   93%     to immutable
0.008188959   03%     normal copy
0.011311666   04%     kotlin toList


1억개
----------------------------------------------------
Seconds       %       Task name
----------------------------------------------------
0.53591875    54%     to immutable
0.225812667   23%     normal copy
0.230502708   23%     kotlin toList

```

만약에 다음과 같이 List에 담기는 객체가 mutable인데 Eclipse Collection의 `Lists.immutable.ofAll()`을 통해 완전한 불변성을 확보했다고 하면 큰 착각이고 성능마저 굉장히 손해를 보는 최악의 선택이 될 수 있다.

```kotlin
    @Test
    fun normal_copy_vs_eclipse_immutable_vs_kotlin_immutable() {
        val kvList: MutableList<KeyValue> = IntStream.rangeClosed(1, 10_000_000)
            .boxed()
            .map { integer -> KeyValue("k$integer", integer) }
            .collect(Collectors.toList())

        val sw = StopWatch("normal vs eclipse immutable vs kotlin immutable")

        sw.start("to eclipse immutable")
        val immutableKeyValues: ImmutableList<KeyValue> = Lists.immutable.ofAll(kvList)
        println(immutableKeyValues.size())
        sw.stop()
        println(immutableKeyValues[0].v)
        immutableKeyValues[0].v = 111
        println(immutableKeyValues[0].v)
        println("---------------------------")

        sw.start("normal copy")
        val mutableIntegersCopied: ArrayList<KeyValue> = ArrayList(kvList)
        println(mutableIntegersCopied.size)
        sw.stop()
        println(mutableIntegersCopied[0].v)
        mutableIntegersCopied[0].v = 222
        println(mutableIntegersCopied[0].v)
        println("---------------------------")

        sw.start("kotlin toList")
        val kotlinxImmutableList: List<KeyValue> = kvList.toList();
        println(kotlinxImmutableList.size)
        sw.stop()
        println(kotlinxImmutableList[0].v)
        kotlinxImmutableList[0].v = 333
        println(kotlinxImmutableList[0].v)
        println("---------------------------")


        println(sw.prettyPrint())
    }

    class KeyValue(
        var k: String,
        var v: Int,
    )


10000000
1
111
---------------------------
10000000
111
222
---------------------------
10000000
222
333
---------------------------
StopWatch 'normal vs eclipse immutable vs kotlin immutable': 0.286756167 seconds
--------------------------------------------------------------------------------
Seconds       %       Task name
--------------------------------------------------------------------------------
0.264592291   92%     to eclipse immutable
0.010511459   04%     normal copy
0.011652417   04%     kotlin toList
```

## 마무리

이쯤에서 결론을 내면 다음과 같다.

>- 컬렉션이나 맵에서 완전한 불변성을 확보하려면 먼저 컬렉션이나 맵에 저장되는 객체가 불변이어야 한다.
>- Eclipse ImmutableCollection/ImmutableMap 은 성능 손실이 생각보다 심하니 주의해서 사용해야 한다.
>- 코틀린 컬렉션이 짱이다 =3=3

