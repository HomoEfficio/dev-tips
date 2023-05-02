# 상속 관계 JPA 매핑

Super - (Sub1, Sub2, Sub3) 관계인 도메인 엔티티가 있다.  
이를 JPA에서 매핑하는 몇 가지 방법과 장단점을 살펴보자.

## @MappedSuperClass

- Super 클래스는 테이블로 생성되지 않고 Sub1, Sub2, Sub3 테이블이 각각 별도로 생성된다.
- Super 테이블이 없으므로 FK를 설정할 수 없기 때문에 다형성을 유지하면서 X:Super = 1:N 인 관계를 구현할 수 없다.
  - ZccVersion:Product = 1:N 이지만 이 Product는 테이블로 생성되지 않아 FK를 가질 수 없으므로 이 관계를 구현할 수 없다.
- 상속 관계지만 다형성 없이 공통되는 필드가 여럿있을 때만 적용 가능
  - 주로 Id, CreatedAt, CreatedBy, LastUpdatedAt, LastUpdatedBy, oVersion 등 모든 엔티티에 사용되는 공통 필드를 포함하는 AbstractEntity에 @MappedSuperClass를 붙여서 사용


## @Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)

- Super 테이블도 생성되며 Sub1, Sub2, Sub3 테이블도 각각 별도로 생성된다.
  - Super 테이블에는 ID와 FK 컬럼만 존재, Sub1, Sub2, Sub3 에 공통되는 필드는 각 개별 테이블의 컬럼으로 생성된다.
- Super 테이블에 FK를 둘 수 있으므로 X:Super = 1:N 인 관계 구현 가능
- x.getSupers()를 실행하면 대략 다음과 같이 복잡한 쿼리가 실행되고 조회 성능이 좋지 않고, Sub가 추가될 수록 점점 악화된다.
    ```
    select setOfAllColumsOfSub1Sub2Sub3
    from Super
    inner join (
      select setOfAllColumsOfSub1Sub2Sub3
      from Super
      union all
      select setOfAllColumsOfSub1Sub2Sub3
      from Sub1
      union all
      select setOfAllColumsOfSub1Sub2Sub3
      from Sub2
      union all
      select setOfAllColumsOfSub1Sub2Sub3
      from Sub3
    ) u on ...
    ```


## @Inheritance(strategy = InheritanceType.JOINED)

- Super 테이블도 생성되며 Sub1, Sub2, Sub3 테이블도 각각 별도로 생성된다.
  - Super 테이블에는 Super, Sub1, Sub2, Sub3 에 공통되는 컬럼이 포함된다
  - Super 테이블에 FK를 둘 수 있으므로 X:Super = 1:N 인 관계 구현 가능
    - X:Super = 1:N 인 관계가 있을 때는 Super 테이블에 X를 참조하는 FK도 포함된다.
- Super 테이블에 Sub1, Sub2, Sub3의 타입을 구분하는 컬럼도 생성된다.
- Sub1, Sub2, Sub3 각 테이블에는 ID와 함께 Sub1, Sub2, Sub3 각각에 고유한 필드만 컬럼으로 생성된다.
  - 대부분의 데이터는 Super 테이블에 저장되고 Sub 테이블에는 일부의 데이터만 포함
- x.getSupers()를 실행하면 union 없이 대략 다음과 같은 쿼리가 실행되므로 Sub가 추가될때마다 left outer join이 추가된다.
    ```
    select setOfAllColumsOfSub1Sub2Sub3, case when ... when ... as type
    from Super
      left outer join Sub1,
      left outer join Sub2,
      left outer join Sub3
    where ...
    ```


## @Inheritance(strategy = InheritanceType.SINGLE_TABLE)

- Super 테이블만 생성되며 Sub1, Sub2, Sub3 테이블은 별도로 생성되지 않고 Super 테이블에 Sub1, Sub2, Sub3 의 모든 컬럼이 저장된다.
  - Super 테이블의 컬럼이 많아지며, Sub1에만 있는 컬럼은 Sub2 레코드에서는 null로 저장
  - Super 테이블에 FK를 둘 수 있으므로 X:Super = 1:N 인 관계 구현. 가능
    - X:Super = 1:N 인 관계가 있을 때는 Super 테이블에 X를 참조하는 FK도 포함된다.
- Super 테이블에 Sub1, Sub2, Sub3의 타입을 구분하는 컬럼도 생성된다.
- Sub1, Sub2, Sub3 의 모든 데이터가 Super 테이블 하나(SINGLE_TABLE)에 저장된다.
- x.getSupers()를 실행하면 union도 없고 join도 없이 대략 다음과 같은 가장 단순한 쿼리가 실행되며 Sub가 추가되어도 영향이 없다.
    ```
    select setOfAllColumnsOfSub1Sub2Sub3
    from Super
    where ...
    ```
## 마무리

- 상속 관계, 연관 관계, 다형성이 모두 필요하다면 MappedSuperClass은 불가 
- TABLE_PER_CLASS는 union을 사용하므로 조회 효율성 매우 낮음
- 각 Sub 마다 고유의 필드가 많다면 정규화의 장점을 살릴 수 있는 JOINED가 적합
- 각 Sub 마다 고유의 필드가 적다면 한 테이블에서 조회 효율을 높일 수 있는 SINGLE_TABLE이 적합


## 참고

- https://thorben-janssen.com/complete-guide-inheritance-strategies-jpa-hibernate/

---
## 마무리 하려고 했는데 이상한 점이..

2022년 11월 현재 Spring Data JPA 2.7.5(Hibernate 5.6.9.Final) 에서는,  
아래와 같이 `DiscriminatorColumn(name = "type")`을 지정해도, 실제 조회 시 날라가는 쿼리를 보면 where 조건에 `type`이 포함돼 있지 않다.

```kotlin
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
```

더 구체적으로는 아래와 같은 관계에서

```kotlin
@Entity
@Table
class OtherEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val oid: Long? = null,
    
    @OneToMany(mappedBy = "otherEntity", cascade = [CascadeType.ALL], orphanRemoval = true)
    val sub1List: MutableList<Super> = mutableListOf(),
    
    @OneToMany(mappedBy = "otherEntity", cascade = [CascadeType.ALL], orphanRemoval = true)
    val sub2List: MutableList<Super> = mutableListOf(),
}

@Entity
@Table
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
abstract class Super(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val oid: Long? = null,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "other_entity_oid")
    private var otherEntity: OtherEntity? = null,
)

@Entity
@DiscriminatorValue("Sub1")
class Sub1(
    oid: Long? = null,
    otherhEntity: OtherEntity?,
) : Super(
    otherEntity = otherEntity
)

@Entity
@DiscriminatorValue("Sub2")
class Sub2(
    oid: Long? = null,
    otherhEntity: OtherEntity?,
) : Super(
    otherEntity = otherEntity
)
```

otherEntity 하나에 Sub1 타입의 엘레먼트가 2개, Sub2 타입의 엘레먼트가 3개라면,
otherEntity.sub1List 에는 Sub1 타입의 엘레먼트 2개만 들어있고,
otherEntity.sub2List 에는 Sub2 타입의 엘레먼트 3개가 들어있을 것 같지만,
실제로는 sub1List 에도 엘레먼트 5개, sub2List 에도 엘레먼트 5개가 들어있다.

저렇게 동작하는 이유는 아마도 sub1List에 대한 mappedBy의 값과 sub2List에 대한 mappedBy의 값이 모두 otherEntity로 동일하기(동일한 컬럼이기) 때문인 것으로 추정된다.

하지만 동일한 컬럼을 사용한다고 하더라도 DB 서버에 전달되는 쿼리의 where 조건에 `type` = 'Sub1' 또는 `type` = 'Sub2' 같은 조건이 포함되면 `type`별로 조회할 수 있는데,  
Hibernate 5.6.9.Final은 where 조건에 `type`을 포함하지 않는다.

그래서 sub1List를 의미대로 Sub1 타입만 포함하는 컬렉션으로 사용하고 싶다면,

- 애초부터 mappedBy 를 각각 구성하거나(이러면 Sub 타입이 추가될 때마다 nullable 컬럼이 추가돼야 하는데 좋은 방향 같지는 않다)
- mappedBy는 동일한 값을 사용하되, 다음과 같이 응용단에서 타입별로 필터링 해서 재구성해서 사용해야 한다.

```kotlin
@Entity
@Table
class OtherEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val oid: Long? = null,
    
    // private val 로 변경해서 Sub1, Sub2 타입을 모두 포함하는 sub1List가 외부로 공개되지 않게 감춘다
    @OneToMany(mappedBy = "otherEntity", cascade = [CascadeType.ALL], orphanRemoval = true)
    private val sub1List: MutableList<Super> = mutableListOf(),
    
    // private val 로 변경해서 Sub1, Sub2 타입을 모두 포함하는 sub1List가 외부로 공개되지 않게 감춘다
    @OneToMany(mappedBy = "otherEntity", cascade = [CascadeType.ALL], orphanRemoval = true)
    private val sub2List: MutableList<Super> = mutableListOf(),
} {
    // 외부에서는 sub1List 대신 sub1List() 사용
    fun sub1List(): MutableList<Sub1> {
        return sub1List.filter {
            it.getType() == MyType.Sub1
        }.map { it as Sub1 }.toMutableList()
    }
    
    // 외부에서는 sub2List 대신 sub2List() 사용
    fun sub2List(): MutableList<Sub2> {
        return sub2List.filter {
            it.getType() == MyType.Sub2
        }.map { it as Sub2 }.toMutableList()
    }
}
```

### 해결!

신동민 님이 알려주신 모범 답안!

Super 쪽에 `@DiscriminatorOptions(force = true)`를 추가하고,  
OtherEntity 쪽에서 다음과 같이 Sub 타입 별로 컬렉션을 구성하면,  

```kotlin
    @OneToMany(mappedBy = "otherEntity", cascade = [CascadeType.ALL], orphanRemoval = true)
    val sub1List: MutableList<Sub1> = mutableListOf(),
    
    @OneToMany(mappedBy = "otherEntity", cascade = [CascadeType.ALL], orphanRemoval = true)
    val sub2List: MutableList<Sub2> = mutableListOf(),
```

where 조건에 `type = 'Sub1'`, `type = 'Sub2'`와 같이 딱 원하는대로 동작한다.

참고로 `@DiscriminatorOptions(force = true)`가 없는 상태에서 위와 같이 Sub 타입 별로 리스트를 구성하면,  
`WrongClassException`이 발생한다.

실무적으로 `@DiscriminatorOptions(force = true)`를 하지 않아야 하는 경우가 잘 없을 것 같은데 `force`의 기본값이 `true`가 아닌 것은 살짝 의아하다.

다른 참고로 `@DiscriminatorOptions(force = true)`는 있지만 아래와 같이 Super 타입으로 컬렉션을 구성하면,

```kotlin
    @OneToMany(mappedBy = "otherEntity", cascade = [CascadeType.ALL], orphanRemoval = true)
    val sub1List: MutableList<Super> = mutableListOf(),
    
    @OneToMany(mappedBy = "otherEntity", cascade = [CascadeType.ALL], orphanRemoval = true)
    val sub2List: MutableList<Super> = mutableListOf(),
```

where 조건에 `type in ('Sub1', 'Sub2')`와 같이 포함되어 null 필터링만 가능할 뿐 Sub 타입별 컬렉션 구성이 되지 않고, sub1List 에 Sub1, Sub2 모두 포함, sub2List 에도 Sub1, Sub2 가 모두 포함된다.

이렇게 상속 관계 매핑 조회에서 핵심적인 역할을 하는 `@DiscriminatorOptions(force = true)`를 보지 못했다.  
특정 기술 관련 문제에서 있을 법한테 인터넷 쉽게 뒤지는 걸로 나오지 않으면, 그냥 없다고 생각하기 전에 공식 문서도 한 번 찾아봐야 한다는 기본 중의 기본을 다시 되새기게 된다.

### 아직 남은 이슈

QueryDSL 관련 이슈가 남아있다.

Q클래스 생성 시 QSub1, QSub2 가 생성되고 그 안에 type 필드도 생긴다.

그래서 조회 시 다음과 같이 조인 문을 사용할 수 있는데,

```kotlin
  .innerJoin(qSub1).on(
      qOtherEntity.oid.eq(qSub1.otherEntity.oid)
          .and(qSub1.type.eq(MyType.SUB1))
```

실제 실행해 보면 Sub1 클래스에 type 이라는 프로퍼티가 없다는 에러가 발생한다.

type은 `@DiscriminatorColumn`에 의해 생겨났을뿐 소스 코드의 프로퍼티로 명시적으로 정의한 게 아니기 때문인 것 같다.

이 문제 해결 방법을 알 수 없어 결국 type이 아닌 다른 값에 의존해서 조건을 사용할 수 밖에 없었다.

(추가) 해결 방법은 매우매우매우매우 간단하다. `@DiscriminatorColumn`으로 지정한 이름의 컬럼을 수퍼클래스 쪽에 명시적으로 필드로 정의하면 된다.

```kotlin
@Entity
@Table
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
abstract class Super(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val oid: Long? = null,
    
    // 여기!! @DiscriminatorColumn(name = "type") 에서 정한 'type' 컬럼에 대한 필드를 명시적으로 지정!!
    @Column(name = "type", insertable = false, updatable = false)
    val myType: MyType,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "other_entity_oid")
    private var otherEntity: OtherEntity? = null,
)
```


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.

