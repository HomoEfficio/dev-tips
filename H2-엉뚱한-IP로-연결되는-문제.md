# H2 엉뚱한 IP로 연결되는 문제

H2의 JDBC URL에 `AUTO_SERVER=TRUE;`를 붙여주면 서버 모드(정확히는 Automatic Mixed Mode)로 동작해서 [여러 프로세스에서 접근 가능](http://www.h2database.com/html/features.html#auto_mixed_mode)하다.

그런데 IP가 자주 바뀌는 노트북에서는, 동일한 JDBC URL로 접근하는 동일한 H2 DB인데도 두 번째로 접근을 시도하는 애플리케이션부터는 다음과 같은 오류가 발생하기도 한다.

```
2019-10-25 22:40:49.351  INFO 10114 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2019-10-25 22:40:55.541 ERROR 10114 --- [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Exception during pool initialization.

org.h2.jdbc.JdbcSQLNonTransientConnectionException: Connection is broken: "java.net.SocketTimeoutException: connect timed out: 218.38.137.27:59191" [90067-199]
	at org.h2.message.DbException.getJdbcSQLException(DbException.java:617) ~[h2-1.4.199.jar!/:na]
	at org.h2.message.DbException.getJdbcSQLException(DbException.java:427) ~[h2-1.4.199.jar!/:na]
  ...
```

위에서 연결을 시도한 `218.38.137.27`는 현재 IP 세팅과는 전혀 상관이 없는 엉뚱한 IP지만, 어디에 저장되어 있는지 몰라도 두 번째 접근 시도부터는 계속 저 IP로 접근을 시도해서 결국 DB 연결에 실패한다.

이 문제를 해결할 수 있는 좋은 옵션이 있으니 바로 `h2.bindAddress`다. [문서](http://h2database.com/html/advanced.html?highlight=bind,address&search=bind%20address#server_bind_address)에도 나와 있듯이 `h2.bindAddress`를 지정하면 항상 고정된 IP로 H2를 바인딩 할 수 있다.

사용법은 다음과 같다.

>1. **DB를 처음 실행할 때 `h2.bindAddress=127.0.0.1`을 옵션으로 주고 실행**하고,  
>1. **이후에 접근 하는 애플리케이션에서도 `h2.bindAddress=127.0.0.1`를 옵션으로 지정하고 실행**하면, 엉뚱한 IP로 연결되는 오류가 발생하지 않고 항상 127.0.0.1로 연결된다.

스프링 클라우드 환경에서 개발용으로 H2를 로컬에서 사용한다면 `h2.bindAddress`는 필수적인 옵션이라고 할 수 있다.
