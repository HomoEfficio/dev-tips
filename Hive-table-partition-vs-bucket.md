# Hive table partition vs bucket

partition 과 bucket 모두 hive 테이블을 물리적으로 분할하는 개념이다.

차이점은 간단하게 말하면 partition 은 디렉터리 분할, bucket 은 파일 분할이다.

partition은 `PARTITIONED BY ()`로 주어진 값을 이름으로 하는 디렉터리를 hdfs 생성해서 분할한다.

bucket 은 partition 에 의해 생성된 디렉터리 내에서 `INTO # BUCKETS` 로 지정된 갯수만큼 파일을 생성하고 `CLUSTERED BY ()`로 주어진 값의 해시값을 기준으로 데이터를 해당 bucket(분할된 여러 파일)에 저장한다.

예를 들어 대략 다음과 같은 DDL 이 있으면

```sql
CREATE TABLE IF NOT EXISTS MY_HIVE_TABLE (
  id string
  , col001 string
  , col002 string)
 PARTITIONED BY ( part_hour string, user_type string )
 CLUSTERED BY ( id )
 SORTED BY ( id ASC )
 INTO 256 BUCKETS
 ROW FORMAT DELIMITED
   FIELDS TERMINATED BY ','
   LINES TERMINATED BY '\n'
 STORED AS INPUTFORMAT
   'org.apache.hadoop.mapred.TextInputFormat'
 OUTPUTFORMAT
   'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
```

hdfs 에서는 다음과 같이 생성된다.

```
/user/hive/warehouse/my_hive_table/part_hour=2020042811/user_type=nba
  ㄴ 000000_0
  ㄴ 000001_0
  ㄴ ...
  ㄴ 000255_0
/user/hive/warehouse/my_hive_table/part_hour=2020042811/user_type=mlb
  ㄴ 000000_0
  ㄴ 000001_0
  ㄴ ...
  ㄴ 000255_0
```



