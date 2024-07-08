# MyBatis, JOOQ, QueryDSL 비교

- MyBatis는 XML에서 텍스트로 쿼리를 구성하므로 type safety의 혜택(오타 감지, 타입 감지, IDE 자동 완성 등)을 누릴 수 없다
- MyBatis와 JOOQ는 RDB에만 사용 가능, QueryDSL은 NoSQL도 지원
  - RDB 쿼리 핸들링에 초점을 맞춰야 한다면 MyBatis, JOOQ가 RDB, NoSQL을 모두 지원하느라 추상 계층이 더 추가된 QueryDSL 보다는 낫고
  - type safety를 고려하면 MyBatis 보다는 JOOQ 가 더 적합하다
- QueryDSL은 JPA와 함께 사용하기 좋다
- type safety를 보장해주는 JOOQ와 QueryDSL은 모두 별도의 파일 생성 과정이 필요하다
- 여기에 아주 잘 정리돼 있다 - https://keencho.github.io/posts/jooq/
