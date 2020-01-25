# Spring Data Batch Insert 최적화

Spring Data JPA가 안겨주는 편리함 뒤에는 가끔 성능 손실이 숨어있다. 이런 손실도 대부분은 올바른 사용법을 적용하지 않았기 때문에 발생하지만, 정말 JPA를 벗어나야만 해결할 수 있는 문제도 있다.

이번에 알아볼 Batch Insert가 그런 예 중 하나다.

# Hibernate의 Batch Insert 한계

[Hibernate 레퍼런스 문서 12.2.1. Batch inserts](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch-insert)의 바로 위에 다음과 같이 **식별자 생성에 IDENTITY 방식을 사용하면 Hibernate가 JDBC 수준에서 batch insert를 비활성화**한다고 나와있다.

>Hibernate disables insert batching at the JDBC level transparently if you use an identity identifier generator.

비활성화하는 이유는 Hibernate 문서에는 없는 것 같아서 다시 찾아보니 StackOverflow에 [Vlad Mihalcea가 올린 댓글](https://stackoverflow.com/a/27732138)에서 단서를 찾을 수 있었다.

>The only drawback is that we can’t know the newly assigned value prior to executing the INSERT statement. This restriction is hindering the “transactional write behind” flushing strategy adopted by Hibernate. For this reason, Hibernates disables the JDBC batch support for entities using the IDENTITY generator.

요는 **새로 할당할 Key 값을 미리 알 수 없는 IDENTITY 방식을 사용할 때 Batch Support를 지원하면 Hibernate가 채택한 flush 방식인 'Transactional Write Behind'와 충돌이 발생하기 때문**에, IDENTITY 방식에서는 Batch Insert를 비활성화 한다는 얘기다. 

그렇다고 IDENTITY 방식말고 SEQUENCE 방식이나 TABLE 방식을 사용하는 것은 더 나쁜 결과를 불러온다. **Batch Insert를 쓸 수 없는 IDENTITY 방식이, Batch Insert를 쓸 수 있는 SEQUENCE 방식이나 TABLE 방식보다 더 빠르다.** 이유는 **SEQUENCE 방식이나 TABLE 방식은 채번에 따른 부하가 상당히 크기 때문**이다. 자세한 내용은 https://github.com/HomoEfficio/dev-tips/blob/master/JPA-GenerationType-별-INSERT-성능-비교.md 를 참고한다.

그럼 그냥 IDENTITY 방식에 머물러야만 하는 것일까? 그래도 Batch Insert가 훨씬 성능이 나을 거라는 믿음으로 Spring Data JPA를 한 번 벗어나봤다.

# Spring Data JDBC

Spring Data에는 JPA만 있는 것이 아니다. https://spring.io/projects/spring-data 에 보면 상당히 다양한 저장소를 지원하는 서브 프로젝트가 많이 있으며, 지금처럼 관계형 데이터베이스에서는 JPA 대신 JDBC를 사용할 수도 있다.

## JdbcTemplate.batchUpdate()

JdbcTemplate에는 Batch를 지원하는 `batchUpdate()` 메서드가 마련돼있다. 여러 가지로 Overloading 돼 있어서 편리한 메서드를 골라서 사용하면 되는데, 여기에서는 batch 크기를 지정할 수 있는 `BatchPreparedStatementSetter`를 사용하는 아래의 메서드를 사용해서 구현해본다.

```java
batchUpdate(String sql, BatchPreparedStatementSetter pss)
```

## 주요 구현 부분

전체 코드는 https://github.com/HomoEfficio/micro-benchmark-spring-boot-batch-insert 여기에서 볼 수 있으며 아주 쉽게 직접 테스트해 볼 수도 있다. 

`ItemJdbc`라는 객체를 `ITEM_JDBC` 테이블에 Batch Insert로 저장한다고 가정하고, 주요 구현부를 살펴보면 다음과 같다. `batchSize` 변수를 통해 배치 크기를 지정하고, 전체 데이터를 배치 크기로 나눠서 Batch Insert를 실행하고, 자투리 데이터를 다시 Batch Insert로 저장한다.

```java
@Repository
@RequiredArgsConstructor
public class ItemJdbcRepositoryImpl implements ItemJdbcRepository {

    private final JdbcTemplate jdbcTemplate;

    @Value("${batchSize}")
    private int batchSize;

    @Override
    public void saveAll(List<ItemJdbc> items) {
        int batchCount = 0;
        List<ItemJdbc> subItems = new ArrayList<>();
        for (int i = 0; i < items.size(); i++) {
            subItems.add(items.get(i));
            if ((i + 1) % batchSize == 0) {
                batchCount = batchInsert(batchSize, batchCount, subItems);
            }
        }
        if (!subItems.isEmpty()) {
            batchCount = batchInsert(batchSize, batchCount, subItems);
        }
        System.out.println("batchCount: " + batchCount);
    }

    private int batchInsert(int batchSize, int batchCount, List<ItemJdbc> subItems) {
        jdbcTemplate.batchUpdate("INSERT INTO ITEM_JDBC (`NAME`, `DESCRIPTION`) VALUES (?, ?)",
                new BatchPreparedStatementSetter() {
                    @Override
                    public void setValues(PreparedStatement ps, int i) throws SQLException {
                        ps.setString(1, subItems.get(i).getName());
                        ps.setString(2, subItems.get(i).getDescription());
                    }
                    @Override
                    public int getBatchSize() {
                        return batchSize;
                    }
                });
        subItems.clear();
        batchCount++;
        return batchCount;
    }
}
```

## 성능 차이

누가 더 빠를까?

물론 Spring Data JDBC로 Batch Insert를 활용하는 방식이 Spring Data JPA에서 IDENTITY 방식으로 키 값을 적용하는 방식보다 빠르다.

그렇다면 얼마나 차이가 날까? 생각보다 크게 차이가 났다.

연관 관계도 없이 단 하나의 엔티티만 저장하는 위 시나리오에서 저장할 데이터 건수를 10,000으로 했을 때 배치 크기 S를 바꿔가면서 테스트 해본 결과 소요 시간(초 단위)은 다음과 같다.

배치 크기(S) | JDBC (A) | Hibernate IDENTITY (B) | (B) / (A)
---|---|---|---
10 | 0.885 |  5.087 |  5.748022599
50 |  0.391 |  4.097 |  10.47826087
100 | 0.356 |  5.218 |  14.65730337
500 | 0.226 |  5.637 |  24.94247788
1000 |   0.216 |  6.241 |  28.89351852
5000 |   0.189 |  5.052 |  26.73015873


## 마무리

>- **아주 많은 수의 데이터를 한 꺼번에 입력할 때는 Spring Data JPA를 잠시 뒤로하고 Spring Data JDBC의 `batchUpdate()`를 활용하는 것이 좋다.**
>
>- **Spring Data JDBC는 Spring Data JPA와 함께 혼용해서 사용할 수도 있으므로, 적재적소에 알맞는 기술을 사용하는 것이 좋겠다.**

