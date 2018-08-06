# JPA 실험실

대충 어설프게 알고 쓰고 있는 JPA. 실험해보며 알아보자.

## 단방향 `@OneToMany`은 특별한 경우가 아니라면 쓰지 않는 것이 좋다.

아래 코드를 보면 알겠지만 객체 관계 관점에서만 바라보면 아주 직관적이고 깔끔 단순한 모델이다. 

`Order`는 `OrderItem`의 존재를 알지만 `OrderItem`은 `Order`의 존재를 모른다.

```java
@Entity
@Table(name = "ORDERS")
@Getter
public class Order {

    @Id
    @GeneratedValue
    @Column(name = "order_id")
    private Long id;
    
    ...
    
    @OneToMany
    private List<OrderItem> orderItems;
    
    ...
}

@Entity
@Table(name = "ORDER_ITEM")
@Getter
public class OrderItem {

    @Id
    @GeneratedValue
    @Column(name = "order_item_id")
    private Long id;

    ...
    
    // Order에 대한 정보 없음
    
    ...
}    
```

- 단방향이므로 `mappedBy` 애트리뷰트를 붙일 대상이 없다.
    - `mappedBy`를 붙일 대상이 없는데도 굳이 붙이면 다음과 같은 에러가 난다.

        >Caused by: org.hibernate.AnnotationException: mappedBy reference an unknown target entity property: io.homo.efficio.cryptomall.entity.order.OrderItem.order in io.homo.efficio.cryptomall.entity.order.Order.orderItems

- 위와 같이 구성 후 테스트 해보면 `ORDERS`, `ORDER_ITEMS` 테이블 외에 다음과 같이 `ORDERS_ORDER_ITEMS`라는 테이블이 생성된다.

    ```
    create table orders_order_items (
       order_order_id bigint not null,
        order_items_order_item_id bigint not null
    ) engine=MyISAM
    ...
    alter table orders_order_items 
       add constraint UK_70a4sa284yptqe6d1xxson8kn unique (order_items_order_item_id)
    ...
    alter table orders_order_items 
       add constraint FKrp82oqw4ek9fpmcf803wxvcta 
       foreign key (order_items_order_item_id) 
       references order_item (order_item_id)
    ...    
    alter table orders_order_items 
       add constraint FK4a5vis32u4bexdg4xyjjc7o4j 
       foreign key (order_order_id) 
       references orders (order_id)    
    ```
- 원했던 것은 단순한 `ORDERS`:`ORDER_ITEM` = 1:N 이었지만, 실제로는 `ORDERS`:`ORDERS_ORDER_ITEMS`:`ORDER_ITEMS` = 1:N:1 관계가 형성된다.
- 이렇게 되면 CUD를 할 때 `ORDER_ORDER_ITEMS`에 대해서도 CUD를 해야하므로 불필요한 오버헤드가 생긴다.

### 정리

> **결국 단방향 `@OneToMany`을 통해 얻고자 했던 단순함도 얻지 못하고 불필요한 오버헤드만 발생하므로 단방향 `@OneToOne`은 별로 좋은 점이 없다.**


## 테스트 메서드에서는 `XXXRepository.save()`만으로는 `flush`가 유발되지 않는다.

```java
@Entity
@Table(name = "PRODUCT")
@Getter
public class Product extends BaseEntity {

    @Id
    @GeneratedValue
    @Column(name = "product_id")
    private Long id;

    private String name;

    private double price;
    
    public Product(@NonNull String name,
                   double price) {
        this.name = name;
        this.price = price;
    }
} 


@RunWith(SpringRunner.class)
@DataJpaTest
public class ProductRepositoryTest {

    @Autowired
    private ProductRepository repository;

    @Test
    public void whenSave__thenReturnProduct() {
        final Product product = repository.save(
                new Product(
                        "어디다쓰 헬스 장갑", 15.00d
                )
        );
    }
}
```

테스트 메서드 실행 후 로그를 보면 아래와 같이 `insert into product`가 실행되지 않음을 알 수 있다.

```
Hibernate: 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
Hibernate: 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
```

하지만, `repository.save()` 후에 `repository.flush()`를 명시적으로 호출하면 아래와 같이 `insert into product`가 실행되고 로그에 표시된다.

```java
    @Test
    public void whenSave__thenReturnProduct() {
        final Product product = repository.save(
                new Product(
                        "어디다쓰 헬스 장갑", 15.00d
                )
        );
        repository.flush();
    }


Hibernate: 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
Hibernate: 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
Hibernate: 
    insert 
    into
        product
        (created_at, last_modified_at, category_id, name, price, product_id) 
    values
        (?, ?, ?, ?, ?, ?)
o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [TIMESTAMP] - [Sun Aug 05 01:12:24 KST 2018]
o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [TIMESTAMP] - [Sun Aug 05 01:12:24 KST 2018]
o.h.type.descriptor.sql.BasicBinder      : binding parameter [3] as [BIGINT] - [null]
o.h.type.descriptor.sql.BasicBinder      : binding parameter [4] as [VARCHAR] - [어디다쓰 헬스 장갑]
o.h.type.descriptor.sql.BasicBinder      : binding parameter [5] as [DOUBLE] - [15.0]
o.h.type.descriptor.sql.BasicBinder      : binding parameter [6] as [BIGINT] - [1]
```

참고로 일반 메서드에서는 `XXXRepository.save()`만으로도 `flush`를 유발한다. 아래 [일반 메서드에 사용되는 `@Transactional`은 `flush`를 유발한다.](#일반-메서드에-사용되는-transactional은-flush를-유발한다) 예제에서 함께 확인할 수 있다.

하지만 **`save()` 이후에 변경된 사항은 명시적으로 `flush()`를 호출해주지 않으면 DB에 반영되지 않으므로 주의**해야 한다.

### 정리

>**테스트 메서드에서는 `XXXRepository.save()`만으로는 `flush`가 유발되지 않으므로, 영속 객체의 데이터에 대한 변경을 모두 마친 후에 명시적으로 `XXXRepository.flush()`를 호출해줘야 한다.**



## 일반 메서드에 사용되는 `@Transactional`은 `flush`를 유발한다.

아래와 같은 CommandLineRunner 구현체로 테스트를 해보면 insert 문은 실행되지만, update 문은 실행되지 않는다.

즉, `save()` 시점에 한 번 `flush`되지만 그 이후에는 영속 상태(MANAGED 상태)인 객체의 데이터에 변경 사항이 생겨도, 명시적으로 `flush()`를 호출하지 않으면 flush 되지 않는다.

```java
@Component
public class InitRunner implements CommandLineRunner {

    @Autowired
    private ProductRepository repository;

    @Override
    public void run(String... args) throws Exception {
        final Product product = repository.save(
                new Product(
                        "어디다쓰 헬스 장갑", 15.00d
                )
        );

        product.setName("나이스 헬스 장갑");
    }
}


insert 
into
    product
    (created_at, last_modified_at, category_id, name, price, product_id) 
values
    (?, ?, ?, ?, ?, ?)
binding parameter [1] as [TIMESTAMP] - [Sun Aug 05 01:42:17 KST 2018]
binding parameter [2] as [TIMESTAMP] - [Sun Aug 05 01:42:17 KST 2018]
binding parameter [3] as [BIGINT] - [null]
binding parameter [4] as [VARCHAR] - [어디다쓰 헬스 장갑]
binding parameter [5] as [DOUBLE] - [15.0]
binding parameter [6] as [BIGINT] - [1]
```

하지만 `run()`에 `@Transactional`을 붙이면 `save()`와 `setName()`을 하나의 트랜잭션으로 묶으면서, 별도의 명시적인 `flush()` 호출 없이도 마지막에 `flush()`를 암묵적으로 호출하게 되므로 update 문도 실행된다.

```java
    @Override
    @Transactional  // <== 추가!!
    public void run(String... args) throws Exception {
        final Product product = repository.save(
                new Product(
                        "어디다쓰 헬스 장갑", 15.00d
                )
        );

        product.setName("나이스 헬스 장갑");
    }
    

insert 
into
    product
    (created_at, last_modified_at, category_id, name, price, product_id) 
values
    (?, ?, ?, ?, ?, ?)
binding parameter [1] as [TIMESTAMP] - [Sun Aug 05 01:42:17 KST 2018]
binding parameter [2] as [TIMESTAMP] - [Sun Aug 05 01:42:17 KST 2018]
binding parameter [3] as [BIGINT] - [null]
binding parameter [4] as [VARCHAR] - [어디다쓰 헬스 장갑]
binding parameter [5] as [DOUBLE] - [15.0]
binding parameter [6] as [BIGINT] - [1]

update
    product 
set
    last_modified_at=?,
    category_id=?,
    name=?,
    price=? 
where
    product_id=?
binding parameter [1] as [TIMESTAMP] - [Sun Aug 05 01:42:17 KST 2018]
binding parameter [2] as [BIGINT] - [null]
binding parameter [3] as [VARCHAR] - [나이스 헬스 장갑]
binding parameter [4] as [DOUBLE] - [15.0]
binding parameter [5] as [BIGINT] - [1]
```

### 정리

>**테스트가 아닌 일반 메서드에 사용되는 `@Transactional`은 트랜잭션을 하나로 묶기 위해 일반 메서드의 종료 시점에 commit 하면서 암묵적으로 `flush`를 유발한다.**


## 테스트 메서드에 사용되는 `@Transactional`은 `flush`를 유발하지 않는다.

하지만 일반 메서드에서와는 달리 테스트 메서드에 사용되는 `@Transactional`은 `flush`를 유발하지 않는다. 게다가 [테스트 메서드에는 `@Transactional`을 붙이지 않아도 기본값으로 롤백을 시켜주므로](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-tx-rollback-and-commit-behavior) 트랜잭션 관리를 별도로 지정할 특별한 사유가 없다면 `@Transactional`을 붙여도 차이가 없으므로 붙일 필요가 없다. 

```java
   ...
   
    @Test
    @Transactional
    public void whenSave__thenReturnProduct() {
        final Product product = repository.save(
                new Product(
                        "어디다쓰 헬스 장갑", 15.00d
                )
        );
        product.setName("나이스 헬스 장갑");
        
        assertThat(product.getName()).isEqualTo("나이스 헬스 장갑");
    }

   ...


// 아래와 같이 sequence 관련 쿼리만 나온다.
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
Hibernate: 
    select
        next_val as id_val 
    from
        hibernate_sequence for update
            
2018-08-05 01:47:44.107 DEBUG 24000 --- [           main] org.hibernate.SQL                        : 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
Hibernate: 
    update
        hibernate_sequence 
    set
        next_val= ? 
    where
        next_val=?
```

따라서, **테스트 메서드에서는 반드시 명시적으로 `flush()`를 호출해줘야 한다. 또는 `flush`를 유발하는 commit, JPQL 실행 등을 해줘야 한다. 그렇지 않으면 위와 같이 사실은 DB에 저장되지 않았음에도 메모리에 저장된 값만으로 비교하므로 테스트는 통과하게 된다.** 이는 로컬 환경에서는 테스트가 통과하지만 실제 운영 환경에서는 실패하는 상황으로 이어질 수 있다.

이에 대해서는 스프링 문서의 [Demonstration of all transaction-related annotations 단원](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-tx-annotation-demo)의 아래 쪽 **Avoid false positives when testing ORM code**에도 특별히 강조되어 있다.

### 정리

>**테스트 메서드에서는 반드시 명시적으로 `flush()`를 호출해주거나, commit, JPQL 쿼리 실행으로 `flush`를 유발해야 한다.**


## 하나의 repository에서만 `flush()`를 호출하면 다른 repository에서의 변경 사항까지 모두 함께 `flush` 된다.

TODO

