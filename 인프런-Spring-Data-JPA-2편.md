# 인프런 Spring Data JPA 2편

## Entity를 클라이언트에 직접 노출하는 것은 안 좋다

- 하나의 Entity라도 여러 api 마다 Entity 필드의 서로 다른 부분 집합을 원할 수 있다.
- 그런데 Entity를 그대로 반환값으로 사용하면 서로 다른 부분 집합 지원 불가
- Entity 필드 이름 변경 시 api 스펙도 변경돼버림
- 외부에 많은 정보를 노출할 수록 변경 어려워짐

## Lazy Loading

- member.getOrders() 까지는 Orders DB에서 조회 하지 않고 Proxy 반환
- member.getOrders().getId() 와 같이 Orders의 필드를 조회하면 실제 DB에서 조회함

## xToOne 시 무한루프

- 1차 문제: 양방향 연관 관계가 있는 엔티티 중 하나를 JSON 으로 변환해서 반환하는 api 를 작성하면 무한루프 발생
- 1차 문제 해결: 한 쪽에 `@JsonIgnore`를 붙여서 JSON 생성시에는 양방향 연관 관계가 발생하지 않도록 조치
- 2차 문제: Hibernate는 LazyLoading을 위한 Proxy 생성 시 ByteBuddy 라이브러리 사용. Jackson이 Serialization 하다가 실객체가 아니라 프록시인 ByteBuddy를 만나면 어쩔 줄을 몰라서 에러 발생
- 2차 문제 해결: Hibernate5Module 빈 등록(jackson-datatype-hibernate 의존 관계 추가 필요)
  - 기본으로는 Lazy 인 애들은 가져오지 않고 null 처리

## DTO 사용

- DTO의 생성자에 Entity 전달
  - DTO -> Entity 의존 관계 발생
  - 덜 중요한 DTO에서 중요한 Entity에 의존하는 것은 큰 문제 아님
- DTO를 사용하고 ToOne 도 Lazy Loading으로 하더라도 N+1(1+N) 문제 발생
  - Order(N) - Member(1) 가 ManyToOne 이더라도 Order 테이블 자체에 Member가 여럿 있으면 1+N 쿼리 발생
  - Order(N) - Member(1) 뿐 아니라, Order(1) - Delivery(1) 까지 있다면 1+N+N 쿼리 발생
- 1+N 쿼리는 JPQL Fetch Join으로 해결 
  - Fetch Join으로 하면 한 방 쿼리로 다 가져오지만, 불필요한 컬럼까지 모두 읽어오는 단점

## 최적화 기본 원칙

- to-One 은 여러 테이블 Join 해도 큰 중복 발생 안 하며, 페이징에도 영향 없음
- to-Many 는 Join 시 1:N 에서 1 쪽 엔티티도 N 행으로 출력되는 중복 발생하며 페이징에도 영향
- 따라서 동일한 엔티티와 연관된 엔티티라도 to-One과 to-Many는 별도의 쿼리로 분리 실행되도록 처리
  - to-One은 연관된 여러 엔티티를 한 방 쿼리로 가져오고
  - to-Many는 Lazy로 하고 실제 읽어올 때 JPQL에 in query 가 사용되도록 파라미터 지정해서 N+1 문제 발생 예방
- 또는 중복 허용하고 한 방 쿼리로 읽어온 후 앱 단에서 1 쪽의 key로 groupingBy 하고 중복 제거하면서 1 쪽에 N을 추가하는 것도 가능하나 row수가 매우 많을 경우 중복에 의한 성능 저하 발생 가능성
