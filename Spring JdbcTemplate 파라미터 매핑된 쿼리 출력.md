# Spring JdbcTemplate 파라미터 매핑된 쿼리 출력

JdbcTemplate에는 대략 `queryFor***(?가포함된쿼리템플릿, 쿼리파라미터, 반환타입)` 이런 메서드들이 있다.

간단하게 다음과 같은 메서드를 작성하면 쿼리 템플릿에 포함된 `?`에 쿼리 파라미터가 매핑된 쿼리문을 확인할 수 있다.

```java
    private String getMappedQuery(String queryTemplate, Object[] queryParams) {
        String[] splitted = queryTemplate.split("\\?");

        StringBuilder sb = new StringBuilder();
        for (int i = 0, len = queryParams.length; i < len; i++) {
            sb.append(splitted[i]).append(getParamString(queryParams[i]));
        }
        if (splitted.length > queryParams.length) {
            sb.append(splitted[splitted.length - 1]);
        }

        return sb.toString();
    }

    private String getParamString(Object param) {
        String result;
        if (param instanceof String) {
            result = "'" + param + "'";
        } else if (param instanceof Integer
                || param instanceof Long
                || param instanceof Float
                || param instanceof Double) {
            result = String.valueOf(param);
        }
        return result;
    }
```
