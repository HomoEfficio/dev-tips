# MySQL JSON Array 를 Space Delimited 문자열로 변환

배열 데이터를 MySQL 에 저장할 때 컬럼 타입을 JSON 으로 하고 다음과 같이 List<String>을 JSON 으로 변환해서 저장하고 사용하면 편리하다.

```java
objectMapper.writeValueAsString(stringList)
```

그런데 JSON 컬럼은 fulltext index에 포함할 수 없으며, 기타 이유로 JSON 컬럼에 저장된 값을 공백 문자열로 변환하여 다른 컬럼이나 테이블에 저장해야 한다면,  
애플리케이션을 통하지 않고도 DB에서 `JSON_EXTRACT`, `JSON_UNQUOTE`, `REGEXP_REPLACE`를 조합하여 다음과 같이 변환할 수 있다.

ex) `["street", "hollywood", "princess"]` => `street hollywood princess`

```sql
(REGEXP_REPLACE(JSON_UNQUOTE(JSON_EXTRACT(JSON컬럼, '$[*]')), '[,\\[\\]\"]', ''))
```


