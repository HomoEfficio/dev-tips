# NotFoundException Pattern

데이터를 어딘가 저장하는 프로그램이라면 `XXXNotFoundException`이 필요하기 마련인데, 엔티티가 없을 때 사용할 예외를 `MemberNotFoundException`, `CourseNotFoundException`, `ProductNotFoundException`, ... 이렇게 `XXXNotFoundException`을 계속 만들어 쓰는 게 영 맘에 들지 않았는데, 아마도 한참 뒷북이겠지만 괜찮은 방법을 알아냈다.

```java
public class EntityNotFoundException extends RuntimeException {

    private final Class<? extends BaseEntity> entityClazz;

    public EntityNotFoundException(Class<? extends BaseEntity> entityClazz) {
        this.entityClazz = entityClazz;
    }

    public EntityNotFoundException(Class<? extends BaseEntity> entityClazz, String message) {
        super(message);
        this.entityClazz = entityClazz;
    }

    public EntityNotFoundException(Class<? extends BaseEntity> entityClazz, Long id) {
        super(String.format("ID [%d] 인 %s 는 존재하지 않습니다.", id, entityClazz.getSimpleName()));
        this.entityClazz = entityClazz;
    }

    public Class<? extends BaseEntity> getEntityClazz() {
        return entityClazz;
    }
}
```

나름 깔쌈하다.

```java
Product product = productRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException(Product.class, id));
```

사용하는 데 문제는 없지만 한 가지 아쉬운 것은 `EntityNotFoundException` 요 이름을 `javax.persistence.EntityNotFoundException`에서 선점했다능..
