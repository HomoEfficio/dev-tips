# Java List 원소 삭제

Java의 List를 순회하면서 특정 조건이 만족될 경우 List의 원소를 다음과 같이 삭제하면, `ConcurrentModificationException`이 발생한다.

```java
for (Element e: elements) {
    if (어쩌구 조건) {
        elements.remove(e);
    }
}
```

우회 방법으로는 List에서 Iterator를 추출해서 Iterator로 순회하면서 삭제하면 된다.

```java
Iterator<Element> iterator = elements.iterator();
while (iterator.hasNext()) {
    Element e = iterator.next();
    if (어쩌구 조건) {
        iterator.remove();
    }
}
```

Java8 에서는 다음과 같이 간단하게 삭제할 수 있다.

```java
elements.removeIf(e -> 어쩌구조건);
```

그런데 만약 그 List가 단순 List가 아니라 다른 엔티티의 연관관계로 존재하는 List라면, 그리고 순회하면서 다른 엔티티에서 삭제하면 위의 방법은 모두 `ConcurrentModificationException`이 발생한다.

그럴 때는 마지막으로 다음과 같은 방법을 쓸 수 있다.

```java
List<Element> elementsToRemove = new ArrayList<>();
for (Element e: elements) {
    if (어쩌구 조건) {
        elementsToRemove.add(e);
    }
}
다른엔티티.removeElements(elementsToRemove);
```
