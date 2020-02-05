# Entity 대신 DTO로 반환

클라이언트에 데이터를 반환할 때는 다음과 같은 이유로 Entity를 그대로 반환하지 말고 DTO로 변환해서 반환하는 것이 좋다.

1. Entity는 해당 Entity의 모든 정보를 담고 있으므로 정보 과노출
  - DTO로 필요한 항목만 골라서 반환하는 것이 좋다.
1. 양방향 연관관계가 있는 Entity는 JSON으로 직렬화하다가 무한루프 발생
  - 이를 해결하기 위해 https://www.baeldung.com/jackson-bidirectional-relationships-and-infinite-recursion 여기 나온대로 `@JsonIgnore`, `@JsonView` 등등을 쓰는 것보다 그냥 DTO를 쓰는 것이 간편하다.
  
특히 2번 항목의 경우 처음에는 양방향 관계가 없어서 그냥 Entity로 반환해도 전혀 문제가 없다가, 나중에 양방향 연관관계가 추가되면 잘 돌던 코드 여기저기서 JSON 직렬화하다가 무한루프가 발생할 수 있다. 그리고 이미 DTO를 써야할 곳에도 Entity가 사용되고 있고, Entity를 써야할 곳에도 Entity가 사용되고 있어서 찾기도 고치기도 힘들다.

따라서 **처음부터, 무조건** DTO를 반환하게 기준을 잡는 것이 좋다.
