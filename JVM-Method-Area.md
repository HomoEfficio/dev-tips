# JVM Method Area

- JVM 명세
  - Method Area 라는 용어만 있고 Method Area의 위치에 대해 강제하는 바 없으며, Perm Gen이나 Metaspace 라는 용어도 없음
    - JVM 스펙에는 메서드 영역이 논리적으로 힙의 일부지만(그래서 가비지 컬렉션 대상이 되지만), 
    - 단순하게는 가비지 컬렉션이나 압축을 하지 않게 구현할 수도 있으며, 
    - 스펙은 메서드의 영역의 위치에 대해 강제하지 않는다고 나와 있기도 하다.
    - 결국 메서드 영역의 위치는 JVM 구현체에 따라 달라질 수 있다.

Perm Gen과 Metaspace는 HotSpot JVM에서 나오는 용어(라고는 하는데 HotSpot JVM 명세 자체를 못 찾겠음 ㅠㅜ)

- JEP 122: Remove the Permanent Generation (https://openjdk.java.net/jeps/122)
  - 기존(8이전)
    - Class Meta 정보와 Interned String, class static 변수가 모두 permanent generation에 저장
    - permanent generation 도 자바 힙의 일부
    - permanent generation 에 저장된 정보도 GC 됨
  - 8
    - Class Meta 정보는 Metaspace라는 새로 신설된 영역에 저장. Metaspace는 OS 힙에 위치
    - Interned String과 class static 변수는 자바 힙에 저장
    - permanent generation 영역은 제거됨
