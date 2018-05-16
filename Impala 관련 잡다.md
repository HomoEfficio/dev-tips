# Impala 관련 잡다

## 테이블 메타 정보 갱신

데이터베이스나 테이블 메타 정보는 보통 다음과 같은 경우에 변경된다.

- Hive: ALTER, CREATE, DROP, INSERT
- Impalad: CREATE TABLE, ALTER TABLE, INSERT

### invalidate metadata

- 하이브와 임팔라가 공유하는 메타스토어에 저장된 정보가 변경되면 임팔라에 의해 캐시되어 있는 정보도 업데이트 되어야 하는데, 이 때 `invalidate metadata`를 실행한다.
- 하나 또는 복수의 테이블의 메타 정보를 무효화(캐시된 메타 정보를 모두 방출)
    - 다음에 해당 테이블에 대한 쿼리를 실행할 때 임팔라가 해당 테이블의 메타 데이터를 새로 reload 한 후에 쿼리를 실행
    - 결국 메타 데이터 갱신을 유발하므로 `invalidate metadata`는 메타 데이터 갱신을 위해 사용
- 하이브 셸에서 테이블을 생성한 후에는 임팔라로 쿼리하기 전에 `invalidate metadata`를 해줘야 한다.
- 하지만 모든 메타 정보의 갱신에 대해 `invalidate metadata`를 실행해야 하는 것은 아니다.
- 실행해야 하는 경우
    - 메타데이터 변경이 발생했고,
    - 그 변경이 클러스터 내의 다른 impalad 인스턴스나 하이브를 통해 발생했고,
    - 그 변경이 임팔라 셸이나 ODBC 같은 클라이언트가 직접 붙는 메타스토어 데이터베이스에 발생했을 때
- 실행하지 않아도 되는 경우
    - ALTER TABLE, INSERT 등 테이블에 수정을 가한 임팔라 노드와 동일한 노드에서 쿼리를 실행할 때
- 대용량 테이블에서는 `invalidate metadata` 실행에 몇 분 정도 소요될 수 있다.
    - 실제 사례
- 비어있는 테이블 A에 하이브 셸에서 insert 한 후, 임팔라 셸에서 `invalidate metadata A`를 하지 않고 해당 테이블을 조회해 보면 조회 결과가 안 나온다.
- `invalidate metadata A` 해준 후 조회하면 결과가 나온다.
- `invalidate metadata` 해준 후 `describe`를 해주면 메타 데이터가 바로 reload 되므로, 첫 쿼리의 응답 시간을 줄여준다.

### refresh

- 기존 테이블에 데이터 파일을 추가한다면 `invalidate metadata`보다 `refresh`가 더 가벼우므로 적합하다.
    - 테이블을 새로 생성했다면 `invalidate metadata`를 해줘야 한다.

>참고
>- http://www.cloudera.com/documentation/cdh/5-1-x/Impala/Installing-and-Using-Impala/ciiu_invalidate_metadata.html
>- https://www.cloudera.com/documentation/enterprise/latest/topics/impala_invalidate_metadata.html

## compute stats

- 테이블 데이터의, 관련된 모든 컬럼과 파티션의 볼륨 및 분포도 정보를 수집
- 수집한 정보는 메타스토어 데이터베이스에 저장되고, 쿼리 최적화에 사용
- 임팔라가 이 정보를 바탕으로 테이블이 큰지 작은지, distinct 값이 많은지 적은지 등을 판단해서 join 쿼리나 insert 연산 시 적절하게 병렬화 적용 가능
- 원래 하이브의 `ANALYZE TABLE` 실행 결과에 의존했으나 사용하기 어렵고 신뢰성이 낮아서 `COMPUTE STATS`로 대체
- 파티션된 테이블에 대해 특정 파티션의 정보만 수집하려면 `COMPUTE INCREMENTAL STATS PARTITION_NAME` 실행
- 동일 테이블에 대해 `COMPUTE STATS`와 `COMPUTE INCREMENTAL STATS`를 모두 실행 금지, 둘 중 하나만 실행해야 함

>참고
>- https://www.cloudera.com/documentation/enterprise/latest/topics/impala_compute_stats.html


## View 연결

`CREATE VIEW VIEW_NAME AS SELECT ... FROM TARGET_TABLE`로 뷰를 만든 후에, `ALTER TABLE TARGET_TABLE RENAME TO TARGET_TABLE_1`와 같이 뷰의 대상 테이블의 이름을 바꾸면 뷰는 이제는 없어진 `TARGET_TABLE`을 계속 참조하므로 에러가 발생한다.

`ALTER TABLE TARGET_TABLE_1 RENAME TO TARGET_TABLE`로 테이블의 이름을 원래 뷰가 참조하고 있던 이름으로 바꾸면, 뷰에 별다른 처리를 하지 않아도 다시 연결이 복원된다.

## 기타 SQL on Hadoop 잡다

- 동시 조회 실행자가 많으면 리소스 사용 증가
- 피닉스는 멀티 세션으로 동시 조회 시 응답 못 주는 경우 있음
- 임팔라도 Like 검색으로 멀티 세션 실행 시 1초 이내 어려울 수 있음
- 임팔라는 `invalidate metadata`를 해줘야 함, 프레스트는 필요없으므로 유리
- 임팔라 + kudu 조합이 프레스토보다는 조금이나마 잘 나온다.
- Kudu는 캐시를 잘 써서 성능 좋고, HDFS는 캐시 개념이 없어서 느리다.
- 프레스토는 C로 되어 있고, 데몬이 떠 있어 잡 실행 환경 조성 비용이 낮아서 빠르고, 하이브는 Java에 데몬 없이 그때그때 잡 실행 환경을 조성하므로 비용이 높아서 느리다.
