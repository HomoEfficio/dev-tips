# SQL on Hadoop 간단 정리

HDFS와 MapReduce의 등장(2006년)으로 적은 비용으로도 대용량 데이터를 분석할 수 있는 시대가 열렸다.

하지만 MR의 사용이 불편하고 처리 속도가 느려서 실시간용으로 사용하기엔 부족했다.

사용성을 개선하기 위해 절차형 스크립트 방식의 Pig와 선언형 SQL 방식의 Hive가 만들어졌지만(2008), 내부적으로 MR을 사용하므로 속도는 여전히 느렸다.

키-값 데이터의 빠른 입출력을 지원하는 HBase의 출현으로 대용량 데이터의 실시간 처리가 가능해졌다.

이후 MR을 사용하지 않고 SQL과 유사한 문법으로 데이터를 처리할 수 있는 다양한 SQL on Hadoop 솔루션이 나왔다.


## Impala

> Real-Time Queries in Apache Hadoop, For Real

- Cloudera에서 [Google의 Dremel 논문](https://research.google.com/pubs/pub36632.html)을 바탕으로 2012년에 아파치 라이선스로 만들고 2017년에 아파치 top-level 프로젝트가 된 SQL on Hadoop 시스템

  ![](https://impala.apache.org/img/impala.png)

- MapReduce를 사용하지 않고 고유의 분산 질의 엔진 사용으로 실시간 질의 가능
- HDFS나 HBase에 저장된 데이터에 대해 SELECT, JOIN, aggregate 함수로 질의 가능
- 분산 질의 엔진은 클러스터 내 모든 데이터 노드에 설치되고 데이터 노드 로컬에서 처리되므로 네트워크 병목 현상 회피 가능
- 메타데이터는 중앙에 저장
- HiveQL 기반이며 모든 HiveQL을 지원하지는 않음
- CPU 부하를 줄인만큼 I/O 대역폭 활용 가능. 순수 I/O bound 질의는 Hive보다 3~4배 정도 빠름
- 복잡한 질의의 경우 Hive는 여러 단계의 MR 또는 Reduce-side 조인 필요, 이 경우 Hive보다 7~45배 정도 빠름
- 분석할 데이터 블록이 파일 캐시되어 있는 경우 Hive보다 20~90배 빠름
- 실시간은 자리에 앉아서 결과를 기다릴 정도의 시간

- 출처
  - http://blog.cloudera.com/blog/2012/10/cloudera-impala-real-time-queries-in-apache-hadoop-for-real/
  - https://impala.apache.org/overview.html
  - http://cidrdb.org/cidr2015/Papers/CIDR15_Paper28.pdf

## Presto

- Facebook에서 2012년에 개발

  ![Imgur](https://i.imgur.com/m4JGZN8.png)

  ![](https://prestodb.io/static/presto-overview.png)

- 페이스북의 300페타바이트 규모의 다양한 데이터소스 대상 실시간 질의 가능
- 지원 파일 포맷
  - Text, SequenceFile, RCFile, ORC and Parquet
  - ORC and Parquet 일 떄 성능이 좋음
- 데이터 처리 과정에서의 중간 데이터를 모두 메모리로 처리
  - 페이스북에서도 16GB heap으로 사용
- 파이프라인 방식으로 데이터 처리
- 다양한 저장소 연결을 지원
  - Cassandra, MySQL, PostreSQL
- Join 기능
  - 이기종 DB 조인
  - Presto 지원 조인 유형


- 출처
  - https://prestodb.io/overview.html
  - http://getindata.com/tutorial-using-presto-to-combine-data-from-hive-and-mysql-in-one-sql-like-query/
  - http://prestodb.rocks/internals/the-fundamentals-types-of-joins-in-sql/
  - https://www.slideshare.net/zhenxiao/presto-uber-hadoop-summit2017

## Drill

## Hive on Tez

- 느린 MR 대신 Tez로 데이터 질의 실행해서 성능 개선
