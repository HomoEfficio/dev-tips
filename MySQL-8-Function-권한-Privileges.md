# MySQL 8 Function

버전 8에서만 발생하는 건지는 모르지만, Function을 정의하기 위해 delimiter 지정만 해도 아래와 같은 에러 발생함

>Caused by: java.sql.SQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'delimiter // // delimiter' at line 1

MySQL shell 에서 root 가 아니지만 All Privileges를 가진 계정으로 다음과 같이 아주 간단한 function create 해도 권한 에러 남

>mysql> create function hello (s char(20)) returns char(50) deterministic return concat('Hello, ', s);  
>-- ERROR 1419 (HY000): You do not have the SUPER privilege and binary logging is enabled (you *might* want to use the less safe log_bin_trust_function_creators variable)

해결 방법은 세 가지다.

1. root 계정으로 해당 DB에 Function 을 생성하고 그 외 계정은 해당 함수를 사용하기만 한다.
1. 사용자 계정에 SUPER 권한을 주고, 해당 사용자 계정으로 Function을 생성한다.
1. `set global log_bin_trust_function_creators = on;` 으로 설정 변경

3번 옵션은 보안 수준이 낮아진다고 하니 가급적 1, 2번으로 처리하는 게 좋겠다.


