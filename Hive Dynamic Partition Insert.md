# Hive Dynamic Partition Insert

## Dynamic Partition Insert란?

보통 Hive에 insert하려면 파티션 컬럼 이름과 값을 지정해야 한다. 이를 Static Partition Insert라 한다.

하지만 **파티션 컬럼 이름만 지정하고 값은 지정하지 않은 채로, 그러니까 값에 따라 동적으로 파티션을 새로 만들기도 하면서 해당 파티션에 insert도 가능**한데 이를 **Dynamic Partition Insert**라 한다.

## Dynamic Partition Insert을 실행하기 위한 전제 조건

Dynamic Partition Insert를 할 때 지켜야할 전제 조건이 몇 가지 있다.

>- 파티션 컬럼들은 select 절의 가장 마지막에 `PARTITION()`으로 정의된 순서에 따라 명시되어야 한다.
>- 모든 파티션 컬럼의 값을 지정하지 않고 Dynamic Partition Insert를 실행하려면 `set hive.exec.dynamic.partition.mode=nonstrict`를 실행해야 한다.
>    - `nonstrict`를 지정하지 않으면 insert 작업 시작 전에 에러가 발생한다.
>- 생성될 최대 파티션의 수를 `set hive.exec.max.dynamic.partitions=10000`와 같이 지정해줘야 한다.
>    - 지정해주지 않으면 기본값 1000 이 적용되며, Dynamic Partition Insert 도중 1000 을 넘는 파티션이 생성되면 에러가 발생한다.
>
>참고: https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DML#LanguageManualDML-DynamicPartitionInserts

`set hive.exec.dynamic.partition.mode=nonstrict`을 지정하지 않고 Dynamic Partition Insert를 실행하면 다음과 같은 에러가 발생한다.

```
StatementCallback; bad SQL grammar [INSERT OVERWRITE TABLE 어쩌구_테이블 PARTITION (seg_id) SELECT 어쩌구_id AS uid, CAST(다른어쩌구_id AS INT) AS 다른어쩌구_id FROM 어쩌구_소스_테이블 WHERE 어쩌구_type = 'cid' DISTRIBUTE BY 다른어쩌구_id ]; nested exception is java.sql.SQLException: Error while compiling statement: FAILED: SemanticException [Error 10096]: Dynamic partition strict mode requires at least one static partition column. To turn this off set hive.exec.dynamic.partition.mode=nonstrict
```

`set hive.exec.max.dynamic.partitions=10000`을 지정하지 않고 Dynamic Partition Insert를 실행하면 다음과 같은 에러가 발생한다.

```
StatementCallback; SQL [INSERT OVERWRITE TABLE 어쩌구_테이블 PARTITION (seg_id) SELECT 어쩌구_id AS uid, CAST(다른어쩌구_id AS INT) AS 다른어쩌구_id FROM 어쩌구_소스_테이블 WHERE 어쩌구_type = 'cid' DISTRIBUTE BY 다른어쩌구_id ]; Error while processing statement: FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.MoveTask; nested exception is java.sql.SQLException: Error while processing statement: FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.MoveTask  
```

특히 이 경우에는 에러 메시지에 별다른 정보가 없어서 디버깅이 힘들다.

## 주의 사항

애플리케이션에서 `set hive.exec.dynamic.partition.mode=nonstrict`와 `set hive.exec.max.dynamic.partitions=10000`를 실행할 때는 **반드시 별개의 `execute()` 메서드를 호출해서 실행해야 한다.**

`set hive.exec.dynamic.partition.mode=nonstrict; set hive.exec.max.dynamic.partitions=10000`와 같이 **한 줄로 붙여서 실행하면 첫 번째 set 문만 실행되고 ; 이후의 두 번째 set문은 실행되지 않는다.**

`set hive.exec.dynamic.partition.mode=nonstrict`을 실행하지 않으면 insert가 실행되면서 금방 에러가 발생하고 에러 메시지에 정보도 제공되므로 문제 해결이 비교적 쉽다. 

하지만 `set hive.exec.dynamic.partition.mode=nonstrict; set hive.exec.max.dynamic.partitions=10000`와 같이 한 문장으로 실행해서 **`set hive.exec.max.dynamic.partitions=10000`가 실행되지 않으면 최대 파티션 갯수 기본값인 1000 개가 넘을 때까지 한참 동안 insert가 진행된 후에 에러가 발생**하고, **에러 메시지에 `return code 1` 외에는 별다른 정보도 주어지지 않아 디버깅도 어려워 진다.**

물론 이 글은.. 주의 사항을 지키지 않아 엄청난 삽질을 했다는 증거다..
