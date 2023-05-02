# MySQL log에서 쿼리 보는 방법

MySQL 5.1 이후에서는 재부팅 없이 설정 변수값 변경만으로 로그 파일에서 쿼리를 확인할 수 있다.

**성능에 악영향을 미칠 수 있으므로 개발용에서만 사용하는 것이 좋다.**

1. 먼저 MySQL Shell에서 현재 설정된 값을 확인해보자.

    ```
    mysql> show variables like 'general_log%';
    +------------------+----------------------------------------+
    | Variable_name    | Value                                  |
    +------------------+----------------------------------------+
    | general_log      | OFF                                    |
    | general_log_file | /var/lib/mysql/XXXXXXXX.log            |
    +------------------+----------------------------------------+
    2 rows in set (0.00 sec)
    ```

1. `general_log`를 `ON`으로 해준다.

    ```
    mysql> set global general_log = 'ON';
    Query OK, 0 rows affected (0.00 sec)
    ```

1. 로그 파일을 tail 로 본다. Heartbeat 성격의 `Query select 1`와 함께 실제 실행된 쿼리를 확인할 수 있다.

    ```
    tail -f /var/lib/mysql/XXXXXXXX.log
    
    2019-10-01T07:10:10.084869Z	 3857 Query	select 1
    2019-10-01T07:10:10.086128Z	 3856 Query	select 1
    2019-10-01T07:10:10.087458Z	 3855 Query	select 1
    2019-10-01T07:10:10.088662Z	 3854 Query	select 1
    2019-10-01T07:10:12.941152Z	 3872 Query	/* select generatedAlias0 from Schedule as generatedAlias0 where generatedAlias0.status=:param0 */ select schedule0_.id as id1_74_, schedule0_.created_date as created_2_74_, schedule0_.last_modified_date as last_mod3_74_, schedule0_.cron_expression as cron_exp4_74_, schedule0_.interval_sec as interval5_74_, schedule0_.schedule_hour as schedule6_74_, schedule0_.schedule_minutue as schedule7_74_, schedule0_.schedule_type as schedule8_74_, schedule0_.status as status9_74_ from schedule schedule0_ where schedule0_.status='RUNNING'
    ```

1. **[매우 중요]** 확인 후 `general_log`를 `OFF`로 돌려 놓는다. `ON`으로 두면 성능에 악영향을 미친다.

    ```
    mysql> set global general_log = 'OFF';
    Query OK, 0 rows affected (0.02 sec)
    ```
