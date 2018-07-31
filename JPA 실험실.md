# JPA 실험실

책에 나온 얘기들 직접 실험해보며 알아보자.

## 단방향 `@OneToMany`

아래 코드를 보면 알겠지만 객체 관계 관점에서는 아주 직관적이고 깔끔 단순한 모델이다. 

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
- 결국 단순함도 얻지 못하고 불필요한 오버헤드만 발생하므로 단방향 `@OneToOne`은 별로 좋은 점이 없다.


    
