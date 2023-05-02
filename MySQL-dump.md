# MySQL 덤프 - mysqldump

데이터를 복제할 필요가 있을 때 mysqldump 를 사용해서 테이블 스키마 생성, 데이터 입력 DDL 을 추출할 수 있다.

간혹 `column_statistics` 관련 오류가 날 때가 있는데, 아래와 같이 `--column-statistics=0`옵션을 추가해주면 오류 없이 수행된다.

```
--column-statistics=0 --result-file="PATH_TO_DUMP_FILE" DATASOURCE
```

아래와 같은 에러가 나면 설명에 나오는대로 `--set-gtid-purged=OFF` 옵션을 추가해주면 된다.

```
Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF.
```


특정 테이블만 mysqldump 로 추출할 때는 `where` 옵션도 사용 가능하다.

```
--where="use_yn = 'Y'" --column-statistics=0 --result-file="PATH_TO_DUMP_FILE" TABLE_NAME
```


