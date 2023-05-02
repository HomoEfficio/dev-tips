# JPA 일대다 단방향 매핑 잘못 사용하면 벌어지는 일

`Parent : Child = 1 : N` 의 관계가 있으면 일대다 단방향으로 매핑하는 것보다 일대다 양방향으로 매핑하는 것이 좋다.


# 조인테이블 방식의 일대다 단방향 매핑

그런데 어떤 특별한 이유가 있을 수도 있고, 그냥 별 생각없이 작성된 레거시 일 수도 있고, 아니면 JPA에 살짝 서툴러서도 있고, 여튼 다음과 같이 직관적으로 단순하게 **`@OneToMany`만 달랑 붙여서 매핑하면 조인테이블 방식의 일대다 단방향 방식으로 매핑된다.**

```java
@Entity
@Getter
public class Parent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Child> children = new ArrayList<>();

    protected Parent() {}

    public Parent(String name) {
        this.name = name;
    }

    public Parent(String name, List<Child> children) {
        this.name = name;
        this.children.addAll(children);
    }
}

@Entity
@Getter
public class Child {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    protected Child() {
    }

    public Child(String name) {
        this.name = name;
    }
}

```

위와 같이 작성하면 조인테이블인 `parent_children`라는 테이블이 새로 생긴다. 뭐 테이블 하나 생기면 어때.. 큰일 나겠어? 라고 생각할 수도 있지만, **`children`이 많지 않을 때만 큰 일이 안 나고, 많으면 제법 큰 일이 난다.**


# 시나리오

위와 같이 매핑된 상태에서 다음과 같은 간단한 시나리오를 생각해보자.

1. `parent`가 10개의 Child를 포함하는 `children`을 가진다.
2. `parent.children`에서 Child의 id가 1, 2인 것 2개만 삭제한다.

1번은 뭐 처음 생성이니 `parent` 1개에 대해 `parent` 테이블에 insert 1회, `children` 10개에 대해 `child` 테이블에 insert 10회 실행된다. 그리고 조인테이블 방식으로 동작하므로 `parent_children` 테이블에도 insert 10회 실행된다.

2번에서 `children` 중에서 2개를 지우므로 `parent_children` 테이블에서 delete 2회 실행되고, `orphanRemoval = true`로 설정되어 있으므로 `child` 테이블에서 delete 2회 실행될 것이다.

하지만 직접 실행해보면 2번은 예상과 완전히 다르게 동작한다!!

```java
@Component
@Transactional
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class OneToManyRunner implements CommandLineRunner {

    @NonNull
    private ParentRepository parentRepository;

    @Override
    public void run(String... args) throws Exception {
        Parent parent1 = new Parent("parent 1");
        for (int i = 1 ; i <= 10 ; i++) {
            parent1.getChildren().add(
                    new Child("child " + i)
            );
        }

        Parent dbParent = this.parentRepository.saveAndFlush(parent1);

        System.out.println("*****************************");
        
        List<Child> children = dbParent.getChildren();
        children.removeIf(child -> 
                child.getId() == 1L && child.getId() == 2L);
    }

}
```

# 실행 결과

`parent_children` 테이블에서 delete 2회, `orphanRemoval = true`로 설정되어 있으므로 `child` 테이블에서 delete 2회 실행될 것으로 예상했지만 실제로는,

- **`parent.children` 10개 모두 delete 되면서 `parent_children` 테이블에서 `children_id`가 1, 2인 것을 제외한 8개의 레코드에 대해 모두 8회의 insert가 실행**되고, 
- 마지막에 `child` 테이블에서 2회의 delete가 실행된다.

```sql
insert into parent (name) values (?)
binding parameter [1] as [VARCHAR] - [parent 1]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 1]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 2]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 3]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 4]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 5]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 6]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 7]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 8]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 9]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 10]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [1]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [2]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [3]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [4]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [5]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [6]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [7]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [8]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [9]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [10]
*****************************
delete from parent_children where parent_id=?  <== 헉!! 형이 왜 여기서 나와!!
binding parameter [1] as [BIGINT] - [1]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [3]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [4]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [5]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [6]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [7]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [8]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [9]
insert into parent_children (parent_id, children_id) values (?, ?)
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [10]
delete from child where id=?
binding parameter [1] as [BIGINT] - [1]
delete from child where id=?
binding parameter [1] as [BIGINT] - [2]
```

앞에서 `children`의 갯수가 많지 않을 때만 큰 일이 안 생긴다고 한 이유가 여기에 있다. 위의 사례에서는 `children`이 10개 밖에 되지 않으므로 insert를 10개 쯤 한다고 해도 사실 거의 티가 나지 않는다. 하지만 10개가 아니라 1000개, 10000개 그 이상이라면? **고작 레코드 2개 삭제하려는 것 뿐인데 1000회, 10000회의 insert가 실행된다.** ㄷㄷㄷ

그런데 왜 이렇게 동작하는 걸까?


# 나름의 사연

실행한 후 `parent_children` 테이블을 보면 다음과 같다.

parent_id | children_id
--- | ---
1 | 3
1 | 4
1 | 5
1 | 6
1 | 7
1 | 8
1 | 9
1 | 10


나: 뭐야, `1 | 1`인 행이랑 `1 | 2`인 행 2개만 지울 수 있었을 것 같은데, 왜 `parent_id`가 1인 걸 몽땅 지워?

Hibernate: 허허.. 그게 말이야.. 허허.. 테이블로 보기엔 저런데.. 허허.. **일대다 단방향이잖아.. 허허.. 그래서.. 허허.. `parent_id`가 1이라는 것을 개별 행에 대한 조건으로 줄 수가 없어..** 허허.. 그래서 `parent_id`가 1인 걸 몽땅 지우고 다시 채웠어.. 허허..

나: 뭐래냐..

이것도 말보다 코드가 더 쉽고 명확한 케이스다. id가 1, 2인 `child`를 삭제하는 코드는 다음과 같다.

```java
List<Child> children = dbParent.getChildren();
children.removeIf(child -> 
        child.getId() == 1L && child.getId() == 2L);  // <-- 여기!!
```

위에 `여기`로 표시한 부분에서 `parent_id`에 대한 조건을 줄 수가 없다. 왜냐고? 위에 Hibernate가 얘기해 준대로 **일대일 단방향이라서 `child`는 `parent`를 모른다. 따라서 `parent_id`를 `children`의 개별 행에 대한 삭제 조건으로 지정할 수가 없다.**

대신에 `dbParent.getChildren()`의 `dbParent`에는 `parent_id`가 1이라는 정보가 있다. 그래서 **`children`를 개별 행 단위로 삭제할 수는 없지만 `parent_children` 테이블에서는 `parent_id`가 1인 행을 모두 삭제할 수는 있다.** 그래서 `parent_id`가 1인 레코드를 모두 delete 한 후에 다시 insert를 반복하는 노가다를 한 것이다.

결국 Hibernate는 주어진 환경에서 최선을 다한 셈이고 아무 죄가 없다. 모두 delete 후 다시 모두 insert 반복으로만 해결할 수 있게 코드를 짠 사람이 잘못이다.


# 해결

이제 문제를 바로잡아보자. 조인테이블 방식의 일대다 단방향 매핑때문에 `children` 쪽에서 행 단위로 `parent_id`를 알 수 없다는 게 원인이었으므로, **어떻게든 `children` 쪽에서 행 단위로 `parent_id`를 알 수 있게 해주면 된다. 즉 테이블 상에서 `children` 쪽에 `parent_id` 컬럼이 추가되도록 매핑하면 된다.**

방법은 두 가지가 있다. 조인테이블이 아닌 조인컬럼 방식의 일대다 단방향 매핑과 일대다 양방향 매핑이다.

먼저 조인컬럼 방식의 일대다 단방향 매핑부터 알아보자.


## 조인컬럼 방식의 일대일 단방향 매핑

이 방식은 단 한 줄의 코드로 쉽게 적용할 수 있다. 물론 예제 코드에서만..

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "parent_id")  // <-- 여기!!
private List<Child> children = new ArrayList<>();
```

위와 같이 Parent 엔티티에 `@JoinColumn(name = "parent_id")`만 추가해주면 된다.

이제 조인테이블 방식이 아니므로 `parent_children` 테이블은 필요 없고, `child` 테이블에 `parent_id` 컬럼이 추가되고, `child` 테이블의 행 단위로 `parent_id`를 알 수 있으므로 몽창 delete 후 몽창 insert 하는 노가다는 발생하지 않고 id가 1, 2인 `child`만 삭제할 수 있을 것이다.

실행해보면 다음과 같다.

```sql
insert into parent (name) values (?)
binding parameter [1] as [VARCHAR] - [parent 1]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 1]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 2]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 3]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 4]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 5]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 6]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 7]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 8]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 9]
insert into child (name) values (?)
binding parameter [1] as [VARCHAR] - [child 10]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [1]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [2]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [3]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [4]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [5]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [6]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [7]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [8]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [9]
update child set parent_id=? where id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [10]
*****************************
update child set parent_id=null where parent_id=? and id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [1]
update child set parent_id=null where parent_id=? and id=?
binding parameter [1] as [BIGINT] - [1]
binding parameter [2] as [BIGINT] - [2]
delete from child where id=?
binding parameter [1] as [BIGINT] - [1]
delete from child where id=?
binding parameter [1] as [BIGINT] - [2]
```

오 역시나 `*****` 아래에 10번의 불필요한 insert 가 모두 사라지고 맨 아래 delete 2회만 실행된 것을 확인할 수 있다.

그런데 `*****` 바로 위에 10번의 update는 또 왜 실행된거지?

이유는 이번에도 단방향이기 때문이다. **조인컬럼 방식으로 변환하면서 `child` 테이블에 `parent_id` 컬럼이 추가되기는 했지만, 단방향이라서 `child`는 `parent`의 존재를 모르므로 `parent_id`의 값을 알 수는 없다.** 뭐랄까 냉장고는 사놨는데 뭘로 채워야할지 모르는..

그래서 **개별 행 단위로는 `parent_id` 컬럼에 값이 없는 채로 insert 되고, insert 된 10개의 행의 `parent_id` 컬럼에는 `dbParent.getChildren()`에서 알아낼 수 있는 `parent_id` 값을 update 를 통해 설정**한다. 하지만 그건 최초에 데이터가 세팅될 때 1회만 그런거고, 그 후에 원하는 레코드만 지울 때는 몽창 지우는 게 아니라 행 단위로 지울 수 있으므로 어쨌든 조인테이블 방식의 문제는 해결한 거라고 할 수도 있겠다.

하지만, 삭제되는 행이 2개가 아니라 수천, 수만이라면? **수천, 수만 회의 delete만 실행되어야 하는데, 수천, 수만의 불필요한 update가 추가로 발생한다.**

**조인테이블 방식에서는 조금만 삭제해도 수 많은 insert 실행이 불필요하게 동반된다는 게 문제였다면, 조인컬럼 방식에서는 많이 삭제하면 수 많은 update가 불필요하게 동반되는 문제로 조금 바뀌었을 뿐 불필요한 오버헤드가 발생한다는 점은 마찬가지다.**

참고로 앞에서 단 한 줄로 적용가능 한 것은 예제 코드라서 가능하다고 했는데, 구체적으로 말하면 `*****` 위에서 update로 값을 자동 세팅해주는 것도 예제 코드라서, `spring.jpa.properties.hibernate.hbm2ddl.auto` 옵션을 `create` 등 마음대로 줄 수 있기 때문에 가능한 것이고, 실 운영 환경에서는 저렇게 수행할 수 없다. 

운영 환경에서는 `child` 테이블에 `parent_id` 컬럼도 직접 추가해줘야 하고 다음과 같이 update 쿼리를 만들어서 **기존에 `parent_children` 테이블에 있던 값을 기준으로 `child` 테이블의 `parent_id` 컬럼에 수동으로 입력해줘야 한다.**

```sql
update child a
set a.parent_id = (
    select b.parent_id 
    from parent_children b
    where a.id = b.children_id
)
```

어쨌든 이번에도 Hibernate는 최선을 다 했다. 사람이 문제지..


## 일대다 양방향 매핑

결국 일대다 단방향 매핑은 insert 든 update 든 오버헤드가 발생할 수 밖에 없다. 가장 깔끔한 답은 양방향 매핑이다.

조인컬럼 방식으로 전환할 때보다는 조금 손이 더 가지만 양은 그리 많지 않다.

```java
@Entity
@Getter
public class Parent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // mappedBy 추가
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Child> children = new ArrayList<>();

    protected Parent() {}

    public Parent(String name) {
        this.name = name;
    }

    public Parent(String name, List<Child> children) {
        this.name = name;
        this.children.addAll(children);
    }
}

@Entity
@Getter
public class Child {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Parent 필드 추가
    @ManyToOne
    @JoinColumn(name = "parent_id")
    private Parent parent;


    protected Child() {
    }

    // 생성자에 Parent 추가
    public Child(String name, Parent parent) {
        this.name = name;
        this.parent = parent;
    }
}

@Component
@Transactional
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class OneToManyRunner implements CommandLineRunner {

    @NonNull
    private ParentRepository parentRepository;

    @Override
    public void run(String... args) throws Exception {
        Parent parent1 = new Parent("parent 1");
        for (int i = 1 ; i <= 10 ; i++) {
            parent1.getChildren().add(
                    new Child("child " + i, parent1)  // 생성 시 parent1 추가
            );
        }
        Parent dbParent = this.parentRepository.saveAndFlush(parent1);

        System.out.println("*****************************");

        List<Child> children = dbParent.getChildren();
        children.removeIf(child -> 
                child.getId() == 1L || child.getId() == 2L);
    }

}
```

실행 결과는 다음과 같다.

```sql
insert into parent (name) values (?)
binding parameter [1] as [VARCHAR] - [parent 1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 1]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 2]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 3]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 4]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 5]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 6]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 7]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 8]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 9]
binding parameter [2] as [BIGINT] - [1]
insert into child (name, parent_id) values (?, ?)
binding parameter [1] as [VARCHAR] - [child 10]
binding parameter [2] as [BIGINT] - [1]
*****************************
delete from child where id=?
binding parameter [1] as [BIGINT] - [1]
delete from child where id=?
binding parameter [1] as [BIGINT] - [2]
```

오! 처음에 원했던 그대로 delete 만 2회 실행될 뿐 아무런 오버헤드도 발생하지 않는다!

## 그래도 일대다 단방향이 비즈 로직에는 딱인데..

딱이긴 한데 상황에 따라 위와 같은 치명적인 단점이 있으니 양방향으로 하는 게 좋다.

그래도 미련이 남는다면 아래와 같은 위로를 전해줄 수 있다.

Child에 있는 Parent에 대한 참조인 parent가 아래와 같이 private으로 선언돼 있고,  

```
    @ManyToOne
    @JoinColumn(name = "parent_id")
    private Parent parent;  // private 이라능!!
```

Child 클래스에 parent에 대한 getter/setter 같은 공개 메서드를 정의하지 않으면,  
바깥에서 볼 때는 Child가 Parent에 대해 전혀 모르는 것처럼 행동하고, 
Parent만 Child에 대해 아는 것처럼 행동하므로,  
코드 작성 관점에서는 일대다 단방향이라고 해도 괜찮지 않을까?

물론 Child를 생성할 때 Parent를 넣어줘야 한다는 사실은 잊지 말자.  
결국 양방향이지만 단방향에 대한 미련을 조금이라도 달래줄수 있지 않을까 해서 덧붙이는 설명이다.


# 정리

>일대다 단방향 매핑은 직관적으로는 단순해서 좋지만,  
>조인테이블 방식은 insert가, 조인컬럼 방식은 update가 오버헤드로 작용한다.
>
>따라서 1:N에서 N이 큰 상황에서는 일대다 단방향 매핑은 사용하지 않는 것이 좋다.
>
>**1:N에서는 웬만하면 일대다 양방향 매핑을 사용하자.**


# 부록 - 응용편

다음과 같이 하나의 Parent에서 2개의 Child에 대해 1:1, 1:N 연관관계 매핑이 필요하면 어떻게 할까?

```java
@Entity
@Getter
public class Parent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // 이게 추가된다면?
    private Child singleChild;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Child> children = new ArrayList<>();

    protected Parent() {}

    public Parent(String name, Child singleChild) {
        this.name = name;
        this.singleChild = singleChild;
    }

    public Parent(String name, Child singleChild, List<Child> children) {
        this.name = name;
        this.singleChild = singleChild;
        this.children.addAll(children);
    }
}

```
이 경우에는 일대일 단방향 매핑을 위해 다음과 같이 Parent 에 `@JoinColumn`을 지정해서 Child를 위한 FK 컬럼을 추가하면, 일대일 단방향 + 일대다 양방향을 함께 쓸 수 있다.

```java
    // 이게 추가된다면?
    //// 일대일 단방향을 쓰되 Child를 가리키는 FK 컬럼을 Parent에 둔다
    @OneToOne
    @JoinColumn(name = "single_child_id")
    private Child singleChild;
```

그럼 `parent` 테이블은 다음과 같이 되고,

`id | name | single_child_id`


`child` 테이블은 다음과 같이 되고, `single_child`와 `children`에 해당하는 데이터가 모두 `child` 테이블에 저장된다.

`id | name | parent_id`

그런데 이렇게 한 테이블에 저장되면 혼동이 될 수도 있을 것 같아 걱정이 된다.

하지만, **일대일 단방향에 의해 저장된 레코드에만 `parent_id` 값이 `NULL`인 상태가 되고,**  
**일대다 양방향에 의해 저장된 레코드에는 `parent_id`에 정상적인 값이 들어가므로 구분 가능**하며 혼동 없이 사용할 수 있다.


