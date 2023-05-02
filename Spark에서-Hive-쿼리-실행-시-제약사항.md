# Spark에서 Hive 쿼리 실행 시 제약사항.md

partition_spec 에 비교 연산자를 사용할 수 없다.

즉, 아래 DDL은 Hive 에서는 실행되지만 Spark 에서는 실행할 수 없다.

```sql
ALTER TABLE my_table
  DROP IF EXISTS PARTITION (part_hour >= '2020042700', part_hour <= '2008042724');
```

이 이슈는 꽤 오래된 것 같은데 https://issues.apache.org/jira/browse/SPARK-14922 여기에 보면 Spark 3.0 에서나 지원될 모양이다.
