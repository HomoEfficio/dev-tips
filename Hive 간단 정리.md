# Hive 간단 정리

## What

- HDFS에 저장된 데이터를 SQL로 쉽게 처리할 수 있는 DW 패키지

### 아키텍처 및 컴포넌트

![https://data-flair.training/blogs/wp-content/uploads/apache-hive-architecture-and-components-of-hive.jpg](https://data-flair.training/blogs/wp-content/uploads/apache-hive-architecture-and-components-of-hive.jpg)

- MetaStore는 실전에서는 주로 MySQL을 사용

## How

### 처리 흐름

![https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/data-processing-workflow-in-hive.jpg](https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/data-processing-workflow-in-hive.jpg)

- MetaStore는 `Impala`, `Presto`, `Drill` 같은 SQL on Hadoop 솔루션에서도 공동 활용

### 데이터 저장 포맷

#### SerDe


#### Text 파일


#### Sequence 파일


#### RC 파일


#### ORC 파일


#### Parquet 파일


  
## Why

- 페이스북에서는 RDBMS만으로는 처리가 어려운 용량의 데이터를 처리
- 처음에는 MR을 사용했으나 프로그래밍이 어렵고 불편
- 스키마 유연성
- 테이블의 portion, bucket 가능
- 하이브 테이블은 HDFS 상에 직접 정의
- JDBC/ODBC 드라이버 사용 가능

## Constraints

### 속도

- MR보다 편리하지만 내부적으로 MR을 사용하므로 MR을 사용하지 않는 다른 SQL on Hadoop 솔루션에 비해 쿼리 속도가 매우 느리다.
- 아래 그림과 같이 Tez, Tez with LLAP로 하면 속도가 많이 개선된다.
  ![https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2016/07/Hive-2.1-blog-MR-vs-Tez-vs-LLAP.png](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2016/07/Hive-2.1-blog-MR-vs-Tez-vs-LLAP.png)
  - https://ko.hortonworks.com/blog/announcing-apache-hive-2-1-25x-faster-queries-much/ 참고

### HiveQL

- HDFS 상에 직접 정의되는 하이브 테이블은 HDFS와 마찬가지로 수정 불가
  - 따라서 `Update`, `Delete` 불가
  - `Insert`도 비어있는 테이블이나 `Insert Overwrite`만 가능
- `View`는 ReadOnly만 가능, 비구체화된 뷰만 가능
- `Having` 절 사용 불가
- 스토어드 프로시저 불가, 대신 MR 스크립트 사용 가능
- 대소문자 구분 안함
- `auto_increment` 미지원
  - https://stackoverflow.com/questions/38949699/hive-auto-increment-after-certain-number 이런 우회 방법이 있기는 함

## Syntax 관련

### Partition

- DDL에서 `PARTITIONED BY (columnA typeA, columnB typeB)`와 같이 지정하면 HDFS 상에 물리적으로 `/user/hadoop/dmp-data-store/columnA=201802/columnB=recopick`와 같이 디렉터리 구조로 데이터가 저장됨
- `PARTITION`으로 지정한 컬럼은 해당 테이블의 새로운 컬럼으로 자동 추가됨

### 데이터 입력

데이터 입력 시 반드시 `PARTITION` 절 사용

```
LOAD DATA LOCAL INPATH '/home/omwomw/data/201802-dmp.csv
OVERWRITE INTO TABLE dmp-data-store
PARTITION (part_month = '201802');

INSERT INTO table_name PARTITION (columnA = '201802', columnB = 'recopick')
SELECT ...

INSERT OVERWRITE table_name PARTITION (columnA = '201802', columnB = 'recopick')
SELECT ...
```

`PARTITION` 절에 컬럼만 명시하고 값을 명시하지 않으면 동적 파티션도 가능하지만, 동적 파티션은 디폴트로 비활성화 되어 있으므로 아래와 같이 따로 활성화 해줘야 한다.

```
hive> set hive.exec.dynamic.partition.mode=nonstrict
hive> set hive.exec.dynamic.partition=true
```

### 데이터 정렬

#### ORDER BY

- 전체 정렬을 하나의 리듀서에서 실행하므로 정렬 대상 데이터가 클 경우 속도 저하

#### SORT BY

- `set mapred.reduce.tasks = 2;`와 같이 리듀서의 갯수를 지정해주면 해당 갯수만큼의 리듀서가 리듀서별로 정렬 실행
- 동일한 컬럼 값을 가진 여러 행이 여러 개의 리듀서에 각각 들어가서 각각 정렬될 수 있으며, 이런 부작용을 방지하기 위해 `DISTRIBUTED BY columnA`를 `SORT BY` 앞에 명시해서 columnA의 값이 같은 행은 같은 리듀서에 들어가게 한다.

#### DISTRIBUTE BY

- `distribute by 컬럼`을 지정하면 리듀서가 해당 컬럼값 기준으로 생성되어 병렬 실행으로 성능 개선 가능
- 다만 `distribute by 컬럼`을 지정했다고 해서 100% 컬럼값 기준으로 병렬 실행이 된다는 보장은 없다.

#### CLUSTERED BY

- `DISTRIBUTED BY`와 `SORT BY`를 함께 사용하는 것과 동일한 기능


### COLLAPSE

- 여러 열을 행으로 처리


## 참고

- https://data-flair.training/blogs/category/hive
- Hive 성능 튜닝: https://docs.treasuredata.com/articles/performance-tuning
