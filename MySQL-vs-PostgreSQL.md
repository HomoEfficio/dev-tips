# MySQL vs PostgreSQL 간단 요약

## TLDR

- CRUD가 많이 발생하는 일반적인 웹 애플리케이션이라면 MySQL로도 충분
- 테라바이트 이상의 대량데이터에서 복잡한 쿼리 연산이 많다면 PostgreSQL이 우위
- 상속, 커스텀타입 등이 필요하다면 ORDBMS인 PostgreSQL이 우위


## 비교표

항목 | MySQL | PostgreSQL | 비고
--- | --- | --- | ---
DBMS 유형 | RDBMS | ORDBMS | PostgreSQL은 [상속](https://www.postgresql.org/docs/15/ddl-inherit.html), [커스텀타입](https://www.postgresql.org/docs/current/rowtypes.html), [함수 오버로딩](https://www.postgresql.org/docs/15/xfunc-overload.html) 가능
연결방식 | Thread per connection | Process per connection | PostgreSQL은 왜 process를 사용하나: [여기](https://wiki.postgresql.org/wiki/Todo#Features_We_Do_Not_Want) 3번째 항목
Update | 저장된 데이터를 변경 | Insert & Delete | PostgreSQL은 새 데이터를 추가하고 구 데이터를 삭제한 후 구 데이터에 연결돼 있던 인덱스나 FK 등을 새 데이터로 업데이트 해야하므로 성능상으로는 불리
Muti Version Concurrency Control | [Undo log 방식](https://mangkyu.tistory.com/53) | [MGA(Multi Generation Architecture) 방식](https://www.slideshare.net/pgday_seoul/mvcc-in-postgre) | 쉽게 생각하면 MySQL은 갱신 전 최종 버전만 Undo log에서 따로 관리, PosgtreSQL은 Insert & Delete이므로 여러 버전을 같은 저장 공간에 두고 포인터로 관리 후 Vacuum으로 불용 기존 레코드 삭제
대소문자 구분 | 구분 없음 | 구분 있음 | -
Materialized View | [지원 안함](https://dev.mysql.com/doc/refman/8.0/en/faqs-views.html#faq-mysql-have-materialized-views) | 지원 함 | -
Partial Index | 지원 안함 | [지원 함](https://www.postgresql.org/docs/current/indexes-partial.html) | 쉽게 인덱스에 조건을 붙일 수 있는 기능이라고 생각하면 되며, 잘 사용하면 인덱스 공간, 관리 비용 절감 가능
Replication | TODO | TODO | -
Clustering | TODO | TODO | -


## 참고

- https://www.integrate.io/ko/blog/postgresql-vs-mysql-the-critical-differences-ko/
- https://techblog.woowahan.com/6550/
- https://www.imaginarycloud.com/blog/postgresql-vs-mysql/
- https://severalnines.com/blog/comparing-data-stores-postgresql-mvcc-vs-innodb/
- https://wiki.postgresql.org/wiki/Replication,_Clustering,_and_Connection_Pooling
- https://blog.panoply.io/postgresql-vs.-mysql

