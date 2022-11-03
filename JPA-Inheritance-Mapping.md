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
    
