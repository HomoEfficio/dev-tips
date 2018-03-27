# Spring JdbcTemplate 사용 시 주의 사항

편리하게 JDBC 프로그램을 짤 수 있게 해주는 스프링의 JdbcTemplate을 너무 편리하게 쓰다보면 SQL Injection의 위험이 있다.

다음 예제를 보자.

```java
String sqlTemplate = "...%s ... %d ...";
String sql = String.format(sqlTemplate, param1, param2, ...);

jdbcTemplate.queryForList(sql);
```

`jdbcTemplate.queryForList(sql)`는 내부적으로 `PreparedStatement`를 만들어서 사용하지 않고 주어진 sql 그대로 `Statement`를 만들어서 사용하므로 SQL Injection 위험이 있다.

그렇다고 너무 걱정할 건 없다. JdbcTemplate API 문서에 나와있는 메서드에 대한 설명 요약 부분을 보면 위험한 메서드를 구별할 수 있다.

![Imgur](https://i.imgur.com/V8AyX46.png)


위 그림에서 빨간색으로 표시한 `Execute a query given static SQL`은 `PreparedStatement`를 사용하지 않고 인자에 있는 문자열 그대로 `Statement`를 만들어 사용한다는 얘기다. 이런 메서드는 SQL Injection 위험이 있다.

녹색으로 표시한 `... to create a prepared statement from SQL`은 `PreparedStatement`를 사용한다는 얘기이므로 SQL Injection으로부터 안전하다.

따라서 Spring JdbcTemplate을 사용할 때는 쿼리 실행 메서드가 `PreparedStatement`를 사용하는 메서드인지 아닌지 확인하고 써야 한다.
