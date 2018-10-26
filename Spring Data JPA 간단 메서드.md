# Spring Data JPA 간단 메서드

## countBy

아래의 두 메서드는 모두 row 수를 반환한다.

```java
@Query(value = "SELECT count(1) FROM target_statistics " +
        "WHERE target_id = :targetId ",
        nativeQuery = true)
Integer findTransferTotalCountByTargetId(@Param("targetId") Long targetId);

Integer countByTargetId(@Param("targetId") Long targetId);
```
