# Effective Java 3e

읽고 기억나는 대로 무작정 되새김질

## 객체 생성과 파괴

### 생성자보다 정적 메서드

- 의미있는 이름
  - from
  - of/valueOf
  - getInstance: 생성할 클래스에서 새 또는 캐시된 인스턴스 반환
  - newInstance: 생성할 클래스에서 새 인스턴스 반환
  - getType: 생성할 클래스가 아닌 다른 클래스에서 팩터리 메서드 정의. 새 또는 캐시된 인스턴스 반환
  - newType: 생성할 클래스가 아닌 다른 클래스에서 팩터리 메서드 정의. 새 인스턴스 반환
- 하위 클래스 반환 가능


### 인스턴스화 방지

- private 생성자


### 다 쓴 객체 참조 해제

- Array[10]에서 0~7까지만 사용한다면 8, 9에는 null을 할당해서 gc 대상이 되게 하라




## 모든 객체의 공통 메서드

### clone


### Autocloseable

- try-with-resources 로 자원 회수 가능


## 클래스와 인터페이스

### 상속보다 조합

- 상속의 폐해: 부모 클래스의 구현 내용에 종속되며 오류 가능성 존재

```java

publc class MySet<E> extends HashSet<E> {
  
  private int addCount = 0;

  @Override
  public boolean add(E e) {
    addCount++;
    return super.add(e);
  }

  @Override
  public boolean addAll(Collection<? extends E> c) {
    addCount += c.size();
    return super.addAll(c);  // super.addAll()은 내부적으로 add()를 호출하도록 구현되어 있는데 이 때 위에서 오버라이드 된 add()가 호출되어 addCount가 두 번 계산된다
  }  
}
```

### 인터페이스는 타입 정의 용도로만 사용

- 상수를 인터페이스에 정의하는 것은 안티패턴. 상수는 인스턴스화 불가능한 클래스에 정의하라


## 제네릭

- raw type 쓰지 말고 `<?>`로 쓰자
