# JPA GenerationType에 따른 INSERT 성능 차이

`GenerationType.AUTO`와 `GenerationType.IDENTITY`의 INSERT 성능은 얼마나 차이날까?

## 테스트 환경

- Spring Boot 2.2.4
- MySQL 5.7.18
- mysql-connector-java 5.1.48

## 관련 코드

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Item {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
//    @GeneratedValue(strategy = GenerationType.AUTO)  // 번갈아가며 테스트
    Long id;

    private String code;
    private String name;
    private String description;
}
```

```java
@Service
@Transactional
@RequiredArgsConstructor
public class ItemService {

    private final ItemRepository itemRepository;

    public void saveAll(Iterable<Item> cryptos) {
        itemRepository.saveAll(cryptos);
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class ItemInitRunner implements CommandLineRunner {

    private final ItemService itemService;

    @Override
    public void run(String... args) throws Exception {
        List<Item> items = new ArrayList<>();
        int N = 10;  // 10, 100, 1000 으로 테스트
        for (int i = 1; i <= N; i++) {
            items.add(new Item(null, "CODE_" + i, "NAME_" + i, "DESC_" + i));
        }
        itemService.saveAll(items);
    }
}
```
## JPA 설정

캐시 영향이 없도록 항상 테이블을 새로 생성해서 수행하도록 hbm2ddl.auto를 create로 하고,  
콘솔 로그에서 쿼리 수행 통계를 볼 수 있도록 설정한다.  
batch_size는 500으로 한다.

```yml
spring.jpa:
  properties.hibernate.hbm2ddl.auto: create
  properties.hibernate.generate_statistics: true
  properties.hibernate.jdbc.batch_size: 500
```

## 테스트 결과

### N = 10

항목 | IDENTITY | AUTO
----|----|----
acquiring JDBC conn # | 1 | 11
acquiring JDBC conn 소요 시간 | 313,501 | 838,810
releasing JDBC conn # | 0 | 10
releasing JDBC conn 소요 시간 | 0 | 154,623
preparing JDBC statements # | 10 | 21
preparing JDBC statements 소요 시간 | 11,669,116 | 9,568,027
executing JDBC statements # | 10 | 20
executing JDBC statements 소요 시간 | 4,769,629 | 6,272,527
executing JDBC batches # | 0 | 1
executing JDBC batches 소요 시간 | 0 | 2,993,124
spent executing 1 flushes 소요 시간 | 5,394,739 | 25,062,101
총 소요 시간 | 22,146,985 | 44,889,212
performing L2C puts # | 0 | 0
performing L2C hits # | 0 | 0
performing L2C misses # | 0 | 0
executing partial-flushes # | 0 | 0

### N = 100

항목 | IDENTITY | AUTO
----|----|----
acquiring JDBC conn # | 1 | 101
acquiring JDBC conn 소요 시간 | 323,767 | 2,255,784
releasing JDBC conn # | 0 | 100
releasing JDBC conn 소요 시간 | 0 | 1,638,315
preparing JDBC statements # | 100 | 201
preparing JDBC statements 소요 시간 | 26,148,309 | 21,349,048
executing JDBC statements # | 100 | 200
executing JDBC statements 소요 시간 | 57,789,351 | 58,588,894
executing JDBC batches # | 0 | 1
executing JDBC batches 소요 시간 | 0 | 19,505,154
spent executing 1 flushes 소요 시간 | 15,774,190 | 159,358,263
총 소요 시간 | 100,035,617 | 262,695,458
performing L2C puts # | 0 | 0
performing L2C hits # | 0 | 0
performing L2C misses # | 0 | 0
executing partial-flushes # | 0 | 0

### N = 1000

항목 | IDENTITY | AUTO
----|----|----
acquiring JDBC conn # | 1 | 1001
acquiring JDBC conn 소요 시간 | 260,083 | 12,268,550
releasing JDBC conn # | 0 | 1000
releasing JDBC conn 소요 시간 | 0 | 14,569,606
preparing JDBC statements # | 1,000 | 2001
preparing JDBC statements 소요 시간 | 156,075,132 | 83,863,894
executing JDBC statements # | 1,000 | 2000
executing JDBC statements 소요 시간 | 737,503,802 | 708,684,620
executing JDBC batches # | 0 | 2
executing JDBC batches 소요 시간 | 0 | 173,789,495
spent executing 1 flushes 소요 시간 | 44,753,467 | 449,931,709
총 소요 시간 | 938,592,484 | 1,443,107,874
performing L2C puts # | 0 | 0
performing L2C hits # | 0 | 0
performing L2C misses # | 0 | 0
executing partial-flushes # | 0 | 0

