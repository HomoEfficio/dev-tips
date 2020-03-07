# QueryDSL group by Date

다음과 같은 테이블에서 날짜인 endDate 기준으로 날짜별 count를 QueryDSL을 사용해서 구하려면?

endDate | value
---- | ----
2020-03-03 01:01:01 | value1
2020-03-03 02:02:02 | value2
2020-03-06 01:01:01 | value1
2020-03-06 02:02:02 | value2
2020-03-06 03:03:03 | value3

검색해보니 아름다운 방법은 없고 `DateTemplate<T>`를 써서 문자열로 바꾼 값으로 groupBy 해야하는 모양이다.

Projection 클래스를 다음과 같이 만들어 두고,

```java
@Getter
@AllArgsConstructor
public class DateCountProjection {

    private LocalDateTime date;
    private Long count;

    public DateCountProjection(String date, Long count) {
        this.date = LocalDate.parse(date).atStartOfDay();
        this.count = count;
    }
}

```

다음과 같이 먼저 `DateTemplate<T>`를 사용해서 문자열(yyyy-MM-dd)로 변환하고, fetch 후에 `Tuple`에서 꺼내서 Projection 객체 생성자에 넣어주면 된다.

```java
public List<DateCountProjection> countByDate() {
    DateTemplate<LocalDateTime> formattedDate = 
        Expressions.dateTemplate(LocalDateTime.class, 
                "DATE_FORMAT({0}, {1})", tbl.endDate, "%Y-%m-%d");

    return from(tbl)
            .groupBy(formattedDate)
            .select(formattedDate, tbl.count())
            .orderBy(formattedDate.asc())
            .fetch().stream()
            .map(tuple -> new DateCountProjection(tuple.get(0, String.class), tuple.get(1, Long)))
            .collect(Collectors.toList());
}

```

마음 같아서는 fetch 이전에 `select()` 안에서 어떻게 해보고 싶었지만 방법을 못 찾았다.

