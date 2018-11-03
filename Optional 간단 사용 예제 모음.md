# Optional 간단 사용 예제 모음

아주 다양한 기능을 제공하는 Optional. 아주 실용적인 것부터 차곡차곡 남겨보자.

## `null` 일 때 새 객체 반환

예를 들어 Map 데이터를 구성하는데 이미 Map에 있는 데이터면 그 데이터를 업데이트하고, 없으면 새 Entry를 만들어서 Map에 추가하는 상황

```java
Map<Long, Object> myMap = fromObj.getMap();

Object value = myMap.get(key);

value = value == null ? new Object() : value;
```

Optional을 사용하면

```java
Map<Long, Object> myMap = fromObj.getMap();

Object value = Optional.ofNullable(myMap.get(key)).orElse(new Object());
```


