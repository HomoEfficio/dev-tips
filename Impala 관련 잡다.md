# Impala 관련 잡다

## 테이블 메타 정보 갱신

데이터베이스나 테이블 메타 정보는 보통 다음과 같은 경우에 발생한다.

- Hive: ALTER, CREATE, DROP, INSERT
- Impalad: CREATE TABLE, ALTER TABLE, INSERT

### invalidate metadata

- 하나 또는 복수의 테이블의 메타 정보를 무효화(캐시된 메타 정보를 모두 방출)
    - 다음에 해당 테이블에 대한 쿼리를 실행할 때 임팔라가 해당 테이블의 메타 데이터를 새로 생성하므로, 결국 메타 데이터 갱신 유발
    - 즉, 메타 데이터 갱신을 위해 사용
- 하이브 셸에서 테이블을 생성한 후에 `invalidate metadata`를 해줘야 한다.

### refresh

- 기존 테이블에 데이터 파일을 추가한다면 `invalidate metadata`보다 `refresh`가 더 가벼우므로 적합하다.
    - 테이블을 새로 생성했다면 `invalidate metadata`를 해줘야 한다.

>참고: http://www.cloudera.com/documentation/cdh/5-1-x/Impala/Installing-and-Using-Impala/ciiu_invalidate_metadata.html

## compute stats

- 테이블 데이터의, 관련된 모든 컬럼과 파티션의 볼륨 및 분포도 정보를 수집
- 수집한 정보는 메타스토어 데이터베이스에 저장되고, 쿼리 최적화에 사용
- 임팔라가 이 정보를 바탕으로 테이블이 큰지 작은지, distinct 값이 많은지 적은지 등을 판단해서 join 쿼리나 insert 연산 시 적절하게 병렬화 적용 가능
- 원래 하이브의 `ANALYZE TABLE` 실행 결과에 의존했으나 사용하기 어렵고 신뢰성이 낮아서 `COMPUTE STATS`로 대체
- 파티션된 테이블에 대해 특정 파티션의 정보만 수집하려면 `COMPUTE INCREMENTAL STATS PARTITION_NAME` 실행
- 동일 테이블에 대해 `COMPUTE STATS`와 `COMPUTE INCREMENTAL STATS`를 모두 실행 금지, 둘 중 하나만 실행해야 함

>참고: https://www.cloudera.com/documentation/enterprise/latest/topics/impala_compute_stats.html

## View 연결

`CREATE VIEW VIEW_NAME AS SELECT ... FROM TARGET_TABLE`로 뷰를 만든 후에, `ALTER TABLE TARGET_TABLE RENAME TO TARGET_TABLE_1`와 같이 뷰의 대상 테이블의 이름을 바꾸면 뷰는 이제는 없어진 `TARGET_TABLE`을 계속 참조하므로 에러가 발생한다.

`ALTER TABLE TARGET_TABLE_1 RENAME TO TARGET_TABLE`로 테이블의 이름을 원래 뷰가 참조하고 있던 이름으로 바꾸면, 뷰에 별다른 처리를 하지 않아도 다시 연결이 복원된다.
