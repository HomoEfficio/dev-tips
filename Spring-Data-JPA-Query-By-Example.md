# Spring Data JPA Query By Example

Spring Data JPA 에는 검색 기능 구현 시 유연하게 사용할 수 있는 [Query By Example](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#query-by-example) 기능이 있다.

쉽게 말하면 **검색 조건을 포함하는 `Example` 객체를 만들어서 Spring Data Repository 의 쿼리 메서드에 인자로 전달하면서 Query By Example 기능을 사용**하게 된다.

## 메커니즘

### Repository

`JpaRepository`는 다음과 같이 `QueryByExampleExecutor`를 상속받고 있고,

```java
public interface JpaRepository<T, ID extends Serializable>
    extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {
```

따라서 다음과 같이 `Example` 객체를 인자로 받는 메서드를 사용할 수 있다.

```java
// 이하 QueryByExampleExecutor
  <S extends T> S findOne(Example<S> example);

  <S extends T> Iterable<S> findAll(Example<S> example);

  <S extends T> Iterable<S> findAll(Example<S> example, Sort sort);

  <S extends T> Page<S> findAll(Example<S> example, Pageable pageable);

  <S extends T> long count(Example<S> example);

  <S extends T> boolean exists(Example<S> example);

// 이하 JpaRepository
  @Override
  <S extends T> List<S> findAll(Example<S> example);

  @Override
  <S extends T> List<S> findAll(Example<S> example, Sort sort);
```

### Example 객체

검색 조건을 담고 있는 `Example` 객체는 대략 다음과 같이 검색어를 포함하는 검색 대상 엔티티 객체(probe)와 조건을 표현하는 ExampleMatcher를 사용해서 만들 수 있다.

```java
// 검색어를 포함하는 검색 대상 엔티티인 probe 생성
Dept dept = new Dept();
dept.setName("c");

Member member = new Member();  // member가 probe 이다
member.setName("m");
member.setDept(dept);

// 검색 조건을 표현하는 ExampleMatcher
ExampleMatcher matcher = ExampleMatcher.matchingAny()  // 모든 matcher를 or 로 연결
  .withMatcher("name", ExampleMatcher.GenericPropertyMatchers.contains().ignoreCase())
  .withMatcher("dept.name", ExampleMatcher.GenericPropertyMatchers.contains())
  .withIgnorePaths("over14", "dept.desc")
  .withIgnoreNullValues();

// 검색 조건을 담고 있는 Example
Example<Member> example = Example.of(member, matcher);
```

`dept.name`과 같이 단순 String 뿐만 아니라 다른 객체에 포함된 필드도 `.` 를 통해 지정할 수 있으므로 편리하다.

위와 같이 만들면 다음 조건을 만족하는 Member를 검색 쿼리가 만들어 진다.

- `name`에 대소문자 불문 m 이 포함
- 또는
- `dept.name`에 대소문자 구분 c 가 포함
- 되는 Member 를 검색하되
- 값이 null 인 필드는 검색 조건에 포함 안 함
- `over14`, `dept.desc`도 검색 조건에 포함 안 함

Query by Example 에서는 검색어가 입력된 객체를 probe 라고 부른다. 앞의 예제 코드에는 member 가 probe 다.

만약에 검색 조건에 회사 설명(`dept.desc`)를 포함하고 싶으면 probe 와 matcher만 다음과 같이 변경하면 된다.

```java
Dept dept = new Dept();
dept.setName("c");
dept.setDesc("d")  // dept.desc 에 사용할 검색어 추가

Member member = new Member();
member.setName("m");
member.setDept(dept);

ExampleMatcher matcher = ExampleMatcher.matchingAny()  // 모든 조건을 or 로 연결
  .withMatcher("name", ExampleMatcher.GenericPropertyMatchers.contains().ignoreCase())
  .withMatcher("dept.name", ExampleMatcher.GenericPropertyMatchers.contains())
  .withMatcher("dept.desc", ExampleMatcher.GenericPropertyMatchers.contains())  // 추가
  .withIgnorePaths("over14") // dept.desc 제거
  .withIgnoreNullValues();

Example<Member> example = Example.of(member, matcher);
```

## 사용

위와 같이 만들어진 `Example` 객체를 사용해서 다음과 같이 쿼리 메서드를 호출할 수 있다.

```java
List<Member> members1 = memberRepository.findAll(example);
Page<Member> members2 = memberRepository.findAll(example, new PageRequest(0, 10, Sort.Direction.DESC, "id"));
```

참고로 `ReactiveQueryByExampleExecutor`도 있으므로 Spring Reactive 에서도 Query By Example 기능을 사용할 수 있다.

## 한계

아쉽지만 제약 사항도 많은 편이다. 레퍼런스 문서에 다음과 같이 나와있다.

>- `or`, `and` 가 중첩되거나 조합된 조건 사용 불가 
>    - No support for nested or grouped property constraints, such as firstname = ?0 or (firstname = ?1 and lastname = ?2).
>
>- 문자열은 starts/contains/ends/regex 조건을 사용할 수 있지만, 그 외의 타입은 정확한 일치 조건만 사용 가능
>    - Only supports starts/contains/ends/regex matching for strings and exact matching for other property types.

사실 위 2 가지 제약 사항만으로도 사용성이 많이 떨어진다. 특히 or/and 를 마음대로 쓸 수 없으면 실제 비즈니스 로직 구현에 꽤 큰 제약이 되기 때문이다.

필드가 많은 엔티티를 검색한다면 다음과 같은 단점도 있다.

- 검색에서 제외할 많은 필드를 모두 `withIgnorePaths()`에 지정 필요
- 검색에 포함하지 않을 모든 필드에 일일이 null 지정 필요
- 

`withIgnorePaths()`에 대해서는 다음과 같은 이슈도 있었다.

### withIgnorePaths(String... paths)

메서드 이름으로 보면 paths 로 지정한 필드들은 검색 조건에 사용되지 않아야 된다.  
즉 실제 쿼리문의 where 조건에도 나타나지 않아야 하는데, 실제로는 타입이 String이 아닌 필드는 paths 에 명시하더라도 where 조건절에 기본값(int = 0, boolean = false)으로 지정돼서 조건에 포함된다.

이 문제를 발생하지 않게 하려면,

- 엔티티에서 int, boolean 인 필드는 Integer, Boolean 로 타입을 지정해줘야 하고,
- probe 를 만들 때, Integer, Boolean로 지정한 필드에 명시적으로 null 값을 지정해주고,
- ExampleMatcher 를 만들 때 `withIgnoreNullValues()`를 호출해서 null 인 필드를 명시적으로 제외해야 한다.


