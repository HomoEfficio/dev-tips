# Spring Data Commons @Reference and MongoDB

몽고디비 컬렉션과 매핑되는 객체의 일부 필드에 `@Reference`라는 애너테이션이 붙어있다.

어디에서 제공되는 애너테이션인가 보니 Spring Data Commons 에서 제공되는 애너테이션인데, 검색 해봐도 뭘 하는 놈인지 당최 알 수가 없..

**무려 [Spring Data Commons 공식 레퍼런스 문서](https://docs.spring.io/spring-data/commons/docs/current/reference/html/#reference)에도 단 한 번도 언급되지 않는다.**

아니 기존 개발자님은 이렇게 베일에 싸인 애너테이션을 어떻게 알고 이걸 사용했을까.. 신기하기 서울역에 그지 없다.

자료로는 알 수 있는 내용이 없어 실험으로 알아낼 수 밖에 없었는데, 다음과 같이 동작한다.


## 가정

- 몽고디비 기준으로 a 컬렉션안에 b 필드(객체)가 있고, b 객체의 bb 필드에 unique 인덱스가 걸려있다.
- 인덱스 자동 생성이 활성화 돼 있다.


## 인덱스 생성

- a.b에 `@Reference`를 붙이지 않으면 중첩 객체인 b에 존재하는 b.bb에 대한 인덱스에 상응하는 a.b.bb 필드에도 b.bb 인덱스와 마찬가지로 unique 인덱스를 생성한다.
- a.b에 `@Reference`를 붙이지 않으면 b.bb 인덱스에 상응하는 인덱스를 생성하지 않는다.


## 중첩 객체 필드 사용

- **`@Reference`를 붙인 객체의 필드는 검색 조건에 사용할 수 없게 된다.**
- `@Reference`를 붙인 객체의 필드를 검색 조건에 사용하면,
  - id 필드라면 dbRef 로 조회하라는 에러 발생
  - id가 아닌 필드라면 org.springframework.data.mapping.MappingException: Invalid path reference user.country! Associations can only be pointed to directly or via their id property! 에러 발생
- Spring Data Repository Query Method 방식, MongoTemplate + Criteria 쿼리 직접 작성, @Query로 Native Query 작성 방식에서 모두 동일한 에러 발생

결국 **중첩 객체 필드를 조회에 사용하려면 `@Reference`를 사용하지 말아야 한다.**


## 정리

- **중첩 객체 필드를 조회에 사용하려면 `@Reference`를 사용하지 말아야 한다.**
- `@Reference`를 사용하지 않으면 인덱스가 자동 생성되면서 데이터 상황에 따라 dup 에러가 날 수 있으며,
  - spring.data.mongodb.auto-index-creation: false 로 지정해서 인덱스 자동 생성을 비활성화 해야 한다.
  - spring data mongodb 2.x 까지는 기본값이 자동 생성 활성화였지만, 3.0부터는 기본값이 비활성화다.
