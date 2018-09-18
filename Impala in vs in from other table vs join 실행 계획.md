# Impala in vs in from other table vs join 실행 계획

- 조회할 컬럼과 조건에 사용되는 컬럼에 인덱스가 걸려있지 않은 상태에서,
- 하나의 테이블에 있는 값만을 조회할 때,
- `in` 의 인자로 들어갈 값을 알고 있어서 `in`에 직접 값을 넣고 조회할 때와, 인자로 들어갈 값이 담긴 테이블을 `join` 할 때의 실행 계획 비교

## in with values

```sql
explain select * from table_a where col01 in ('a', 'b', 'c', 'd', 'e');
OK
Max Per-Host Resource Reservation: Memory=0B
Per-Host Resource Estimates: Memory=2.62GB

PLAIN-ROOT SINK
|
01: EXCHANGE [UNPARTITIONED]
|
00: SCAN HDFS [dbXXX.table_a]
    partitions=25254/25254 files=31069 size=887.23GB
    predicate: uid IN ('a', 'b', 'c', 'd', 'e')
time taken: 0.400 seconds, fetched: 10 rows
```

## in with values from other table

```sql
explain select * from table_a where col01 in (select col01 from table_b where col02 % 2 = 0);
OK
Max Per-Host Resource Reservation: Memory=35.00B
Per-Host Resource Estimates: Memory=4.69GB
WARNING: The following tables are missing relevant table and/or column statistics.
dbXXX.table_b

PLAIN-ROOT SINK
|
04: EXCHANGE [UNPARTITIONED]
|
02: HASH JOIN [LEFT SEMI JOIN BROADCAST]
|   hash predicates: col01 = col01
|   runtime filters: RF000 <- col01
|
|--03: EXCHANGE [BROADCAST]
|  |
|  01: SCAN HDFS [dbXXX.table_b]
|      partitions=1/1 files=1 size=4.11MB
|      predicates: col02 % 2 = 0
|
00: SCAN HDFS [dbXXX.table_a]
    partitions=25254/25254 files=31069 size=887.23GB
    runtime filters: RF000 -> uid
time taken: 0.327 seconds, fetched: 22 rows
```

## join

```sql
explain select * from table_a a join table_b b on a.col01 = b.col01 where b.col02 % 2 = 0;
OK
Max Per-Host Resource Reservation: Memory=35.00B
Per-Host Resource Estimates: Memory=4.69GB
WARNING: The following tables are missing relevant table and/or column statistics.
dbXXX.table_b

PLAIN-ROOT SINK
|
04: EXCHANGE [UNPARTITIONED]
|
02: HASH JOIN [INNER JOIN BROADCAST]
|   hash predicates: a.col01 = b.col01
|   runtime filters: RF000 <- b.col01
|
|--03: EXCHANGE [BROADCAST]
|  |
|  01: SCAN HDFS [dbXXX.table_b b]
|      partitions=1/1 files=1 size=4.11MB
|      predicates: b.col02 % 2 = 0
|
00: SCAN HDFS [dbXXX.table_a a]
    partitions=25254/25254 files=31069 size=887.23GB
    runtime filters: RF000 -> a.uid
time taken: 0.343 seconds, fetched: 22 rows
```

## 해석

- `join` 이 RDBMS에서는 인덱스 여부에 따라 나을 수도 있지만, Hive에서는 MR이 한 번 더 유발된다.
- 단일 테이블 조회고 `in` 이 사용되는 컬럼에 인덱스가 걸려 있다면 `in`이 무조건 빠르다.
