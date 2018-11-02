# DDD Start - 4장 JPA

## Spring Data JPA

- 책에는 JPA, JPQL을 이용해서 Spring 기반의 프로젝트는 대부분 Spring Data JPA를 사용
- dmp-backend 도 Spring Data JPA 기반
- JPA 구현체는 하이버네이트
- EntityManager 추상화
- 메서드 이름 규칙으로 쿼리문 자동 생성
  - https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation
- Repository를 직접 구현해야 할 일이 줄어듬

## 매핑 구현

### 엔티티, 밸류 기본 매핑 구현

p109 그림 4.2

실제 코드 설명

### 기억할 키워드

- `@Entity`
- `@Embeddable`, 중첩 가능, setter 쓰지 말자
- `@Embedded`
- `@AttributeOverrides`, `@AttributeOverride`

### 기본 생성자

- `@Entity`와 `@Embeddable`로 매핑된 클래스의 객체 생성 시 하이버네이트는 기본 생성자를 사용
- setter를 안 쓰는 밸류 객체는 기본 생성자가 필요 없고 있어서도 안 되지만 JPA 구현체에 따라 기본 생성자 필요
- 기본 생성자를 `private`로 선언해서 오용을 막을 수 있으나, 
  - JPA에서는 Lazy Loading을 위해 객체를 상속해서 만드는 프록시를 사용하며,
  - 이 프록시를 통해 부모 객체인 엔티티/밸류 객체를 생성하므로 `protected`로 선언해야 함

### 필드 접근 방식

- 하이버네이트는 `@Id`나 `@EmbeddedId`가 필드에 위치하면 필드 접근 방식, 메서드에 위치하면 메서드 접근 방식
- 대부분 필드 접근 방식 사용

### AttributeConverter

실제 사용 코드 보기

### 밸류 컬렉션 객체를 별도의 테이블로

- `@ElementCollection`
- 옵션: `@CollectionTable`, `@OrderColumn`

Order -> List<OrderItem>

### 밸류 컬렉션 객체를 하나의 컬럼으로

- `@Convert`
- 구분자를 사용해서 join 하고 하나의 문자열로

### 밸류 객체를 아이디로 사용

- `@EmbeddedId`
- 아이디에 특수한 기능 추가 가능
- `equals()`, `hashcode()`

### 밸류 객체를 별도의 테이블에 저장

- 의미적으로 부모 역할을 하는 클래스에 `@SecondaryTable`로 밸륙 객체가 저장될 테이블 설정
- 그냥 Product, Review 로 설명하면 좋았을 걸 난데없이 Article, ArticleContents로 설명해서 아쉬움
- `articleRepository.findOne(1L)` 실행 시 Article 테이블과 `@SecondaryTable`로 설정한 테이블을 JOIN해서 조회 -> 5장에서 설명

### 밸류 컬렉션을 `@Entity`로 매핑

- 밸류 객체는 중첩은 지원하지만 상속은 지원하지 않음
- 상속 구조가 존재하는 밸류 객체는 불가피하게 `@Entity`로 매핑해야 함
- 상속 구조의 표현
  - 부모 클래스: `@Entity`
    - `@Inheritance`: 보통 `InheritanceType.SINGLE_TABLE` 사용
    - `@DiscriminatorColumn`: 자식 타입을 구분할 수 있는 컬럼
  - 자식 클래스: `@Entity`
    - `@DiscriminatorValue`: 부모 클래스에서 `@DiscriminatorColumn`로 지정한 컬럼에 들어갈 구분 문자열 지정
- 밸류 컬렉션뿐 아니라 그냥 밸류 객체도 `@Entity`로 매핑해야 함
- 엔티티 간 연관 관계 설정은 `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`
- 책 설명이 부족하므로 김영한 님의 JPA 책 참고

### ID 참조와 조인 테이블 이용한 단방향 M:N

- 애그리거트 간 연관 관계를 ID 참조 방식 + 조인 테이블 방식으로 하면 애그리거트 삭제 시 조인 테이블의 데이터만 삭제 되므로 안전

- 3장 참고

## 애그리거트 로딩 전략

### 개념

- 지연 로딩(LAZY): 연관된 엔티티나 밸류 객체를 프록시로 조회(참조값만 가져옴)
- 즉시 로딩(EAGER): 연관된 엔티티나 밸류 객체를 즉시 조회(실제값까지 모두 조회)

### 기본 전략

- `@ManyToOne`, `@OneToOne`, 즉 `@*ToOne`: 즉시 로딩
- `@OneToMany`, `@ManyToMany`, 즉 `@*ToMany` 와 `@ElementCollection`: 즉시 로딩
- 가능하면 모든 연관 관계에 지연 로딩을 활용하자
- `@*ToMany`에 즉시 로딩 적용 시 주의 사항
  - 컬렉션을 2개 이상 로딩하면 cartesian product 발생, JPA가 필터링하긴 하지만 성능에 부정적 영향
  - 컬렉션 즉시 로딩 시 항상 외부 조인이 사용됨


## 애그리거트 영속성 전파

- 저장, 삭제 시 애그리거트의 일관성 유지
- 밸류 객체는 영속성이 전파됨
- 엔티티 객체는 `CascadeType`으로 지정해줘야 함

## 식별자 생성 기능

- 보통 자동 생성되는 대리 키 방식 사용
- Spring Boot 2 부터는 기본 방식이 바뀌어 `GenerationType.IDENTITY`로 해줘야 함
