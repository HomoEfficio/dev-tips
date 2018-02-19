# 일라스틱서치가 왜 빠른가?

일라스틱서치는 MySQL 같은 데이터베이스가 아니고, 아파치 Lucene 기반의 검색 엔진을 기반으로 하는 JSON 문서 저장소다.

일라스틱서치는 "Hello to the world"라는 문자열을 ["Hello", "to", "the", "world"]로 토크나이징해서 인덱스하기도 하고 중요한 단어인 ["Hello", "world"]만을 토크나이징해서 인덱스하는 등 유연하게 다양한 방식으로 인덱스를 생성해서 Full-text 검색에 특히 뛰어나다.

예를 아래와 같은 데이터는

```
Doc1
"class": {
    "name": "history",
    "lecturer": "Thomas"
}

Doc2
"class": {
    "name": "mathmatics",
    "lecturer": "Feynmann"
}

Doc3
"class": {
    "name": "physics",
    "lecturer": "Feynmann"
}

Doc4
"class": {
    "name": "art",
    "lecturer": "Thomas"
}
```

일라스틱서치에 다음과 같이 저장된다.

text | Document
----|----
Thomas | Doc1, Doc4
physics | Doc3
Feynmann | Doc2, Doc3
... | ...


일라스틱서치는 조인이나 서브쿼리를 사용할 수 없으므로, 
하나의 쿼리로 처리하지 못 하고 여러 차례의 쿼리를 실행해야하며 성능이 떨어진다(양이 많다면 이런 경우에도 다른 DB보다 빠를 수도 있다). 

기본적으로 HTTP API 서버로서 동작하고 별도의 전용 클라이언트가 없다.

질의 방식도 SQL을 지원하지 않고 다음과 같은 JSON을 이용한 방식으로 다소 낯설다.

```
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": { "balance": { "order": "desc" } }
}
```

또한 저장된 값을 수정할 때도 성능이 떨어진다.

역시나 용도에 맞게 사용하는 것이 중요하다.
