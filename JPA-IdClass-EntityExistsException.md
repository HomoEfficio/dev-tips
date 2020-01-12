# JPA-EntityExistsException

다음과 같이 TraitTarget이라는 Entity가 있다. `TraitTargetSourceIdType`와 1:N 관계를 가지고 있다.

```java
@Entity
public class TraitTarget {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "traitTarget")
    @LazyCollection(LazyCollectionOption.FALSE)
    private List<TraitTargetSourceIdType> sourceIdTypes;
    
    ...
}
```

`TraitTargetSourceIdType`는 다음과 같이 `@IdClass`로 복합키를 사용한다.

```java
@Entity
@IdClass(TraitTargetSourceIdTypeId.class)
public class TraitTargetSourceIdType {

    @Id
    @ManyToOne(optional = false)
    @JoinColumn(name = "trait_target_id")
    private TraitTarget traitTarget;

    @Id
    @ManyToOne(optional = false)
    @JoinColumn(name = "id_type")
    private IdType idType;

    @Convert(converter = BooleanToStringConverter.class)
    private boolean selected;
    
    ...
}
```

복합키 클래스인 `TraitTargetSourceIdTypeId`는 다음과 같이 `Serializable`을 구현해야하고, `equals()`와 `hashCode()`도 구현해야 한다.

```java
public class TraitTargetSourceIdTypeId implements Serializable {

    private Long traitTarget;
    private String idType;

    public TraitTargetSourceIdTypeId() {
    }

    public TraitTargetSourceIdTypeId(Long traitTarget, String idType) {
        this.traitTarget = traitTarget;
        this.idType = idType;
    }

    @Override
    public boolean equals(Object o) {
        ...
    }

    @Override
    public int hashCode() {
        ...
    }
}
```

이 상태에서 Transaction 시작 없이 `TraitTarget`을 저장하면,

```java
...
TraitTarget traitTarget = traitTargetRepository.findById(traitTargetId).orElseThrow(() -> new RuntimeException());
...
// @Transactional 도 없고, PlatformTransactionManager 로 Tx 설정도 없는 상태에서
traitTarget.setXXX(xxx);
traitTargetRepository.save(traitTarget);
```

다음과 같이 IdType의 식별자 값이 `gaid`인 데이터가 이미 세션에 존재한다는 에러가 난다. 뭔가 의도하지 않은 insert가 발생한다는 것 같다.

>javax.persistence.EntityExistsException: A different object with the same identifier value was already associated with the session : [a.b.c.d.IdType#gaid]

검색해보면 주로 `entityManager.persist()` 대신 `entityManager.merge()`를 사용하면 위 에러가 발생하지 않는다고 하는데,  
Spring Data를 사용하면 그냥 `repository.save()`로 작성하면 내부적으로 `entityManager.persist()`와 `entityManager.merge()`를 알아서 구분해서 실행해주므로, 명시적으로 `entityManager.merge()`를 호출할 필요는 없다.  
굳이 `merge()`를 호출한다해도 이 경우(`@Transactional` 도 없고, `PlatformTransactionManager` 로 Tx 설정도 없는 상태)에는 여전히 동일한 에러가 발생한다.

일반적인 경우라면, 그러니까 `@Transactional`이나 `PlatformTransactionManager`로 Tx 설정한 상태라면,  
TraitTarget을 저장하는데 IdType이 저장될 필요는 없고, 실제로 저장되지도 않는다. 따라서 IdType에 대해 insert도 발생하지 않으며, 위와 같은 `EntityExistsException` 에러도 발생하지 않는다.

**이 문제는 `@Transactional`이나 `PlatformTransactionManager`로 Tx 설정을 해주면 해결**된다.

특히 `PlatformTransactionManager`를 사용할 때는 다음과 같이 **Tx 시작 이후에 `TraitTarget`를 새로 조회하고 그 결과를 저장해야 에러가 발생하지 않는다.**  
이 때 **commit 이나 rollback 을 누락하면 DB 테이블 레코드에 Lock이 걸려 해제되지 않을 수 있으므로 주의**해야 한다.

```java
@Autowired
private PlatformTransactionManager transactionManager;
...
    TraitTarget traitTarget = traitTargetRepository.findById(traitTargetId).orElseThrow(() -> new RuntimeException());
    ...
    TransactionStatus transactionStatus = transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
        TraitTarget dbTraitTarget = traitTargetRepository.findById(traitTarget.getId()).orElseThrow(() -> new RuntimeException());
        dbTraitTarget.setXXX(xxx);
        traitTargetRepository.save(traitTarget);
    } catch (Exception e) {
        transactionManager.rollback(transactionStatus);
        throw new RuntimeException(e);
    }
    transactionManager.commit(transactionStatus);
  ```
```


