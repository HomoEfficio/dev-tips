# JPA 필요한 것만 조회하자

JPA 는 편리하지만 편리함 뒤에 숨어있는 성능 손실 위험이 있다. 이건 JPA가 그 자체로 성능 상 불리하다는 얘기가 아니라, 편하게만 쓰다보면 잘못 쓰는 길로 빠져서 성능에 해를 끼칠 위험도 꽤 있다는 얘기다.

여러가지 원칙이 있겠지만, 이번에 기억해둬야 할 원칙은 **JPA는 필요한 것만 조회하자**

## 엔티티

아래는 어떤 카테고리를 나타내는 엔티티다. 카테고리는 보통 하위에 동일한 타입의 다른 카테고리를 자식으로 가지고 있고, 상위에도 동일한 타입의 다른 카테고리를 부모로 가질 수 있다. `parent_category_id` 컬럼에는 인덱스가 걸려있고, 데이터는 양 54,000건 있다.

```java
public class ZZZCategory extends BaseEntity implements TreeEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne
    @JoinColumn(name = "parent_category_id")
    private ZZZCategory parentCategory;

    @OneToMany(mappedBy = "parentCategory", cascade = CascadeType.PERSIST)
    private List<ZZZCategory> childCategories = new ArrayList<>();

    @OneToMany(mappedBy = "ZZZCategory", cascade = CascadeType.PERSIST)
    private List<ZZZ> ZZZs = new ArrayList<>();
}
```


## 현재 조회 메서드

여러 개의 카테고리ID가 주어지면 주어진 카테고리ID들과 그들의 하위 카테고리ID를 모두 가져오는 로직이 필요한데 다음과 같이 구현돼 있다.

```java
    private Set<Long> getAllZZZCategoryIdsIncludingDescendants(Set<Long> ZZZCategoryIds) {
        List<ZZZCategory> ZZZCategories = ZZZCategoryRepository.findAll(ZZZCategoryIds);
        Set<Long> ids = new HashSet<>();
        ids.addAll(ZZZCategoryIds);
        ids.addAll(getChildIds(ZZZCategories));
        return new HashSet<>(ids);
    }

    private Set<Long> getChildIds(List<ZZZCategory> ZZZCategories) {
        Set<Long> ids = new HashSet<>();
        if (ZZZCategories == null)
            return ids;

        ids.addAll(ZZZCategories.stream().map(ZZZCategory::getId).collect(toList()));

        for (ZZZCategory category : ZZZCategories) {
            List<ZZZCategory> children = ZZZCategoryRepository.findWithFetchedChildren(category.getId()).getChildCategories();
            ids.addAll(getChildIds(children));
        }

        return ids;
    }
```

전체 약 5.4만건에서 21개의 ID로 그 하위 ID를 모두 조회하니 약 4.8만건이 조회된다. 소요 시간은 200초 내외다.

특별할 건 없지만 필요한 건 ID 뿐인데, 편한 맛에 `ZZZCategoryRepository.findAll(ZZZCategoryIds)`를 사용해서 카테고리를 통째의 목록을 읽어와서 ID 를 추출하는 방식이라는 게 눈에 살짝 거슬린다. 이걸 개선해보자.


## 개선 후 조회 메서드

ZZZCategory 엔티티 통째로 조회하지 않고 ID만 조회하는 방식으로 바꿨다.

```java
    private Set<Long> getAllZZZCategoryIdsIncludingDescendants(Set<Long> ZZZCategoryIds) {
        Set<Long> ids = new HashSet<>();
        ids.addAll(ZZZCategoryIds);
        ids.addAll(getChildIds(ZZZCategoryIds));

        return new HashSet<>(ids);
    }

    // ID를 인자로 받도록 변경
    private Set<Long> getChildIds(Set<Long> ZZZCategoryIds) {
        Set<Long> ids = new HashSet<>();
        if (ZZZCategoryIds == null) {
            return ids;
        }

        ids.addAll(ZZZCategoryIds);

        for (Long ZZZCategoryId : ZZZCategoryIds) {
            // ID 리스트를 반환하는 메서드로 대체
            List<Long> childCategoryIds = ZZZCategoryRepository.findChildCategoryIds(ZZZCategoryId);
            ids.addAll(getChildIds(new HashSet<>(childCategoryIds)));
        }

        return ids;
    }
```

엔티티를 통째로 가져오는 `ZZZCategoryRepository.findAll(ZZZCategoryIds)` 대신에 ID만 가져오는 메서드를 QueryDSL로 구현해서 대신 사용했다. 앞에서 얘기한 것처럼 `parent_category_id`에는 인덱스가 걸려 있으므로 조회 효율이 괜찮을 것이다.

```java
    @Override
    public List<Long> findChildCategoryIds(Long parentId) {
        QTraitCategory traitCategory = QTraitCategory.traitCategory;
        return from(traitCategory)
                .where(traitCategory.parentCategory.id.eq(parentId))
                .select(traitCategory.id)
                .fetch();
    }
```

개선 후 실행해보니 똑같은 조회를 수행하는데 소요 시간이 **200초 내외에서 25초 내외로 개선되어 약 8배 정도 빨라졌다.**

사실 위 엔티티 코드는 간단한 설명을 위해 실제보다 축약한 형태라서 위 코드만으로는 8배 까지 개선되지는 않을 수도 있다.

여튼 중요한 것은 **JPA를 쓸 때는 편하다고 다 읽어들이지 말고 조금 손이 가더라도 필요한 것만 조회하자.**

