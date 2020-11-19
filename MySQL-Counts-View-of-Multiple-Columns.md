# MySQL Counts View of Multiple Columns - 테이블의 컬럼 조건별 카운트를 보여주는 뷰

테이블의 여러 컬럼의 컬럼값 별 카운드가 필요할 때가 있다.

다음과 같은 뷰를 만들면 사용하기 편하다.

```
CREATE
VIEW `user_counts_by_status_v` AS
SELECT
  1 AS `id`
  , sum(1) AS `total`
  , sum(`active` = 'Y' and `suspended` = 'N') AS `active`
  , sum(`active` = 'N' and `suspended` = 'Y') AS `suspended`
  , sum(`approved` = true) AS `approved`
FROM `user`
WHERE `deleted` = 'N'
```
