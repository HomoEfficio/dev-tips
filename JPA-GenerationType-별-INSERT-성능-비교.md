# JPA GenerationType에 따른 INSERT 성능 차이

가장 널리 사용되는 MySQL에서 `GenerationType.AUTO`와 `GenerationType.IDENTITY`의 INSERT 성능은 얼마나 차이날까?

# 테스트 환경

- Spring Boot 2.2.4
- MySQL 5.7.18
- mysql-connector-java 5.1.48

# 관련 코드

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Item {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
//    @GeneratedValue(strategy = GenerationType.AUTO)  // IDENTITY와 번갈아가며 테스트
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

캐시의 영향이 없도록 실행할 때마다 애플리케이션과 MySQL을 재구동해서 테스트한다.

```java
@Component
@RequiredArgsConstructor
public class ItemInitRunner implements CommandLineRunner {

    private final ItemService itemService;

    @Override
    public void run(String... args) throws Exception {
        List<Item> items = new ArrayList<>();
        int N = 10;  // 10, 100, 1000, 10000 으로 테스트
        for (int i = 1; i <= N; i++) {
            items.add(new Item(null, "CODE_" + i, "NAME_" + i, "DESC_" + i));
        }
        itemService.saveAll(items);
    }
}
```

# JPA 설정

- 항상 테이블을 새로 생성해서 캐시 영향이 없도록 `hbm2ddl.auto`를 `create`로 설정
- 콘솔 로그에서 쿼리 수행 통계를 볼 수 있도록 `generate_statistics`를 `true`로 설정
- batch insert를 활성화하기 위해 `jdbc.batch_size`는 500으로 설정

```yml
spring.jpa:
  properties.hibernate.hbm2ddl.auto: create
  properties.hibernate.generate_statistics: true
  properties.hibernate.jdbc.batch_size: 500
```

---
# 테스트 결과

>**MySQL에서는 IDENTITY 방식이 빠르다.**
>
>- IDENTITY 방식이 총 소요 시간 기준으로 대략 1.5 ~ 2.5배 가량 빠르다.
>- AUTO 방식에서는 batch insert가 실행됨에도 불구하고 채번 부하가 상당히 커서, batch insert가 실행되지 못 하는 IDENTITY 방식보다 느리다.
>- 특히 insert 하려는 데이터 row 수가 커넥션 풀에 있는 커넥션보다도 많다면, 해당 테이블의 auto_increment 키 값은 IDENTITY 방식으로 생성하는 것이 좋다.
>
>따라서, 단지 batch insert가 성능에 유리하다는 점만을 고려해서 AUTO를 선택하는 것은 좋지 않은 선택일 수 있다.
>
>다만 **MySQL이 아닌 다른 DB에서는 다른 결과가 나올 수 있다.**

총 소요 시간(nanoseconds) 기준 

N | IDENTITY | AUTO | AUTO / IDENTITY
----|----|----|----
10 | 22,146,985 | 44,889,212 | 2.03
100 | 100,035,617 | 262,695,458 | 2.63
1000 | 938,592,484 | 1,443,107,874 | 1.54
10000 | 42,020,329,144 | 66,472,754,040 | 1.58


# 분석

[MySQL 5.7 레퍼런스 문서](https://dev.mysql.com/doc/refman/5.7/en/insert-optimization.html) 에 따르면 insert 시 단계별 비중은 다음과 같다.

- Connecting: (3)
- Sending query to server: (2)
- Parsing query: (2)
- Inserting row: (1 × size of row)
- Inserting indexes: (1 × number of indexes)
- Closing: (1)

즉, **기본적으로는 연결과 전송의 비중이 꽤 크며, insert 하려는 행의 갯수가 많을 수록 전체적으로는 실제 데이터 입력에 소요되는 비중이 큼을 알 수 있다.**


## 연결 부문

N | IDENTITY | AUTO | AUTO / IDENTITY
----|----|----|----
10 | 313,501 | 993,433 | 3.17
100 | 323,767 | 3,894,099 | 12.03
1000 | 260,083 | 26,838,156 | 103.19
10000 | 299,656 | 195,476,471 | 652.33

Hibernate 통계 로그에 따르면 이유는 모르지만 **AUTO 방식의 경우 N + 1 개의 커넥션이 사용**된다. 연결/해제에 소요되는 시간이 N = 100 일 때는 10배, N = 1000 일 때는 100배, N = 10000 일 때는 650배에 이른다. 스프링 부트 2.X에서 기본으로 사용되는 [HikariCP의 기본 maximumPoolSize는 10](https://github.com/brettwooldridge/HikariCP#frequently-used)이므로 N = 100, N = 1000, N = 10000 인 경우는 훨씬 더 많은 시간이 소요된 걸로 보인다.


## JDBC statements 부문

N | IDENTITY | AUTO | AUTO / IDENTITY
----|----|----|----
10 | 16,438,745 | 18,833,678 | 1.15
100 | 83,937,660 | 99,443,096 | 1.18
1000 | 893,578,934 | 966,338,009 | 1.08
10000 | 41,908,124,798 | 62,465,642,852 | 1.49

### IDENTITY 방식

아래 MySQL 로그에 나와있지만, IDENTITY 방식은 예상대로 `id` 외의 값만 VALUES 에 포함되며 **N회의 insert만 준비/실행된다.** 그리고 Hibernate 통계 로그에 따르면, **`jdbc.batch_size` 를 지정했음에도 불구하고 IDENTITY 방식에서는 batch insert는 실행되지 않았다.** 

이유는 [Hibernate 레퍼런스 문서 12.2.1. Batch inserts](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch-insert) 바로 위에 나와있는 것처럼, **식별자 생성에 IDENTITY 방식을 사용하면 Hibernate가 JDBC 수준에서 batch insert를 비활성화하기 때문**이다. 

결국 **Hibernate에서 IDENTITY 방식으로 식별자를 생성하면 batch insert는 사용할 수 없다.**

### AUTO 방식

반면에 AUTO 방식은 채번 테이블을 통해 구한 `id` 값도 VALUES 에 포함되며, 채번 1회마다 select, update 2회의 JDBC statements가 실행되므로 **채번에만 2N개의 JDBC statements가 실행**된다. 실제 데이터는 batch insert를 통해 입력되므로 ceil(N/batch_size)회의 batch insert가 실행되는 걸로 통계에 잡힌다. 

결국 **총 2N + ceil(N/batch_size) 회의 JDBC statements가 실행**된다.

한 가지 특이한 점은 AUTO 방식 사용시 `batch_size`가 지정돼있으면, Hibernate 통계 로그 상으로는 항상 batch insert가 실행되는 것으로 나오지만, MySQL 로그 상으로는 최초에 `order_inserts` 설정을 명시하지 않았을 때는 `insert into item (code, description, name, id) values ()`, `insert into item (code, description, name, id) values ()`, ... 와 같이 N회의 insert 문을 실행했다. 그런데 `order_inserts`를 true로 설정하면 `insert into item (code, description, name, id) values (),(),(),(),(), ...`와 같이 하나의 insert 문으로 `batcn_size`개의 row를 batch insert 하고, 그 후에는 다시 `order_inserts`를 false로 설정해도 `insert into item (code, description, name, id) values (),(),(),(),(), ...`와 같이 batch insert 문을 실행한다.

최초의 케이스만 특이했다고 치면 **AUTO 방식을 사용하고 `batch_size`를 명시하면 기본적으로 batch insert는 활성화 된다**고 보면 되겠다.


## flush 부문

N | IDENTITY | AUTO | AUTO / IDENTITY
----|----|----|----
10 | 5,394,739 | 25,062,101 | 4.65
100 | 15,774,190 | 159,358,263 | 10.10
1000 | 44,753,467 | 449,931,709 | 10.05
10000 | 111,904,690 | 3,811,634,717 | 34.06

IDENTITY 방식과 AUTO 방식 모두 1회의 flush만 발생한다. 하지만 소요 시간은 5 ~ 34배 가량 차이가 발생한다. 이유는 **AUTO 방식일 때는 채번 과정까지 포함해야하므로 flush 될 내용이 많기 떄문인 것으로 보인다.**


---
# 테스트 상세 결과

## Hibernate 통계

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

### N = 1,000

항목 | IDENTITY | AUTO
----|----|----
acquiring JDBC conn # | 1 | 1,001
acquiring JDBC conn 소요 시간 | 260,083 | 12,268,550
releasing JDBC conn # | 0 | 1,000
releasing JDBC conn 소요 시간 | 0 | 14,569,606
preparing JDBC statements # | 1,000 | 2,001
preparing JDBC statements 소요 시간 | 156,075,132 | 83,863,894
executing JDBC statements # | 1,000 | 2,000
executing JDBC statements 소요 시간 | 737,503,802 | 708,684,620
executing JDBC batches # | 0 | 2
executing JDBC batches 소요 시간 | 0 | 173,789,495
spent executing 1 flushes 소요 시간 | 44,753,467 | 449,931,709
총 소요 시간 | 938,592,484 | 1,443,107,874
performing L2C puts # | 0 | 0
performing L2C hits # | 0 | 0
performing L2C misses # | 0 | 0
executing partial-flushes # | 0 | 0

### N = 10,000

항목 | IDENTITY | AUTO
----|----|----
acquiring JDBC conn # | 1 | 10,001
acquiring JDBC conn 소요 시간 | 299,656 | 87,713,781
releasing JDBC conn # | 0 | 10,000
releasing JDBC conn 소요 시간 | 0 | 107,762,690
preparing JDBC statements # | 10,000 | 20,001
preparing JDBC statements 소요 시간 | 2,360,763,185 | 1,291,847,077
executing JDBC statements # | 10,000 | 20,000
executing JDBC statements 소요 시간 | 39,547,361,613 | 60,828,259,387
executing JDBC batches # | 0 | 2
executing JDBC batches 소요 시간 | 0 | 345,536,388
spent executing 1 flushes 소요 시간 | 111,904,690 | 3,811,634,717
총 소요 시간 | 42,020,329,144 | 66,472,754,040
performing L2C puts # | 0 | 0
performing L2C hits # | 0 | 0
performing L2C misses # | 0 | 0
executing partial-flushes # | 0 | 0 


## MySQL Log, 편의상 N = 3

MySQL 로그 보는 방법은 https://github.com/HomoEfficio/dev-tips/blob/master/MySQL-log에서-쿼리보는-방법.md 참고

### IDENTITY

batch insert 실행 안 되고 N회의 insert만 실행

```
Time                 Id Command    Argument
2020-01-22T15:48:47.893291Z  4699 Query SET autocommit=0
2020-01-22T15:48:47.938707Z  4699 Query insert into item (code, description, name) values ('CODE_1', 'DESC_1', 'NAME_1')
2020-01-22T15:48:47.944258Z  4699 Query insert into item (code, description, name) values ('CODE_2', 'DESC_2', 'NAME_2')
2020-01-22T15:48:47.948299Z  4699 Query insert into item (code, description, name) values ('CODE_3', 'DESC_3', 'NAME_3')
2020-01-22T15:48:47.959162Z  4699 Query commit
2020-01-22T15:48:47.959825Z  4699 Query SET autocommit=1
```

### AUTO

채번 과정 당 select, update 각 1회씩 실행되며, 최초에 `order_inserts`를 설정하지 않았을 때 batch insert 실행 안 됨

```
Time                 Id Command    Argument
2020-01-22T15:47:44.518399Z  4676 Query SET autocommit=0
2020-01-22T15:47:44.536334Z  4677 Query SET autocommit=0
2020-01-22T15:47:44.546640Z  4677 Query select next_val as id_val from hibernate_sequence for update
2020-01-22T15:47:44.547597Z  4677 Query update hibernate_sequence set next_val= 2 where next_val=1
2020-01-22T15:47:44.548063Z  4677 Query commit
2020-01-22T15:47:44.549387Z  4677 Query SET autocommit=1
2020-01-22T15:47:44.558830Z  4677 Query SET autocommit=0
2020-01-22T15:47:44.560170Z  4677 Query select next_val as id_val from hibernate_sequence for update
2020-01-22T15:47:44.561436Z  4677 Query update hibernate_sequence set next_val= 3 where next_val=2
2020-01-22T15:47:44.562029Z  4677 Query commit
2020-01-22T15:47:44.562604Z  4677 Query SET autocommit=1
2020-01-22T15:47:44.563314Z  4677 Query SET autocommit=0
2020-01-22T15:47:44.565509Z  4677 Query select next_val as id_val from hibernate_sequence for update
2020-01-22T15:47:44.566982Z  4677 Query update hibernate_sequence set next_val= 4 where next_val=3
2020-01-22T15:47:44.568003Z  4677 Query commit
2020-01-22T15:47:44.568507Z  4677 Query SET autocommit=1
2020-01-22T15:47:44.587060Z  4676 Query select @@session.tx_read_only
2020-01-22T15:47:44.587475Z  4676 Query insert into item (code, description, name, id) values ('CODE_1', 'DESC_1', 'NAME_1', 1)
2020-01-22T15:47:44.588132Z  4676 Query insert into item (code, description, name, id) values ('CODE_2', 'DESC_2', 'NAME_2', 2)
2020-01-22T15:47:44.588425Z  4676 Query insert into item (code, description, name, id) values ('CODE_3', 'DESC_3', 'NAME_3', 3)
2020-01-22T15:47:44.590900Z  4676 Query commit
2020-01-22T15:47:44.591423Z  4676 Query SET autocommit=1
```

채번 과정 당 select, update 각 1회씩 실행되며, `order_insert` 적용 시 batch insert 실행됨

```
Time                 Id Command    Argument
2020-01-23T02:22:44.077300Z  6145 Query SET autocommit=0
2020-01-23T02:22:44.116656Z  6146 Query SET autocommit=0
2020-01-23T02:22:44.137712Z  6146 Query select next_val as id_val from hibernate_sequence for update
2020-01-23T02:22:44.185856Z  6146 Query update hibernate_sequence set next_val= 2 where next_val=1
2020-01-23T02:22:44.188451Z  6146 Query commit
2020-01-23T02:22:44.190763Z  6146 Query SET autocommit=1
2020-01-23T02:22:44.209690Z  6146 Query SET autocommit=0
2020-01-23T02:22:44.211387Z  6146 Query select next_val as id_val from hibernate_sequence for update
2020-01-23T02:22:44.213568Z  6146 Query update hibernate_sequence set next_val= 3 where next_val=2
2020-01-23T02:22:44.215175Z  6146 Query commit
2020-01-23T02:22:44.216865Z  6146 Query SET autocommit=1
2020-01-23T02:22:44.218185Z  6146 Query SET autocommit=0
2020-01-23T02:22:44.220182Z  6146 Query select next_val as id_val from hibernate_sequence for update
2020-01-23T02:22:44.222653Z  6146 Query update hibernate_sequence set next_val= 4 where next_val=3
2020-01-23T02:22:44.230562Z  6146 Query commit
2020-01-23T02:22:44.233993Z  6146 Query SET autocommit=1
2020-01-23T02:22:44.352658Z  6145 Query select @@session.tx_read_only
2020-01-23T02:22:44.363424Z  6145 Query insert into item (code, description, name, id) values ('CODE_1', 'DESC_1', 'NAME_1', 1),('CODE_2', 'DESC_2', 'NAME_2', 2),('CODE_3', 'DESC_3', 'NAME_3', 3)
2020-01-23T02:22:44.376553Z  6145 Query commit
2020-01-23T02:22:44.377581Z  6145 Query SET autocommit=1
```

채번 과정 당 select, update 각 1회씩 실행되며, `order_insert` 적용 후 다시 해제 시 batch insert 실행됨

```
Time                 Id Command    Argument
2020-01-23T02:30:12.317200Z  6344 Query SET autocommit=0
2020-01-23T02:30:12.348454Z  6345 Query SET autocommit=0
2020-01-23T02:30:12.363988Z  6345 Query select next_val as id_val from hibernate_sequence for update
2020-01-23T02:30:12.366240Z  6345 Query update hibernate_sequence set next_val= 2 where next_val=1
2020-01-23T02:30:12.368774Z  6345 Query commit
2020-01-23T02:30:12.370192Z  6345 Query SET autocommit=1
2020-01-23T02:30:12.386549Z  6345 Query SET autocommit=0
2020-01-23T02:30:12.388022Z  6345 Query select next_val as id_val from hibernate_sequence for update
2020-01-23T02:30:12.390362Z  6345 Query update hibernate_sequence set next_val= 3 where next_val=2
2020-01-23T02:30:12.391718Z  6345 Query commit
2020-01-23T02:30:12.393100Z  6345 Query SET autocommit=1
2020-01-23T02:30:12.394377Z  6345 Query SET autocommit=0
2020-01-23T02:30:12.395446Z  6345 Query select next_val as id_val from hibernate_sequence for update
2020-01-23T02:30:12.397046Z  6345 Query update hibernate_sequence set next_val= 4 where next_val=3
2020-01-23T02:30:12.398607Z  6345 Query commit
2020-01-23T02:30:12.400178Z  6345 Query SET autocommit=1
2020-01-23T02:30:12.431752Z  6344 Query select @@session.tx_read_only
2020-01-23T02:30:12.434311Z  6344 Query insert into item (code, description, name, id) values ('CODE_1', 'DESC_1', 'NAME_1', 1),('CODE_2', 'DESC_2', 'NAME_2', 2),('CODE_3', 'DESC_3', 'NAME_3', 3)
2020-01-23T02:30:12.438186Z  6344 Query commit
2020-01-23T02:30:12.440464Z  6344 Query SET autocommit=1
```
