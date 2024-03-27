# 캐시, 1차 캐시, 2차 캐시

- 1차 캐시, 2차 캐시는 보통 JPA나 그 구현체에서 사용되는 용어다.

## 캐시

- **통상적으로 말하는 캐시는 일반적으로 메서드 수준에서 동작**하며, 캐시 데이터는 인메모리 캐시를 사용한다면 하나의 애플리케이션 인스턴스 수준에서 유지되고, 분산 캐시를 사용한다면 분산 캐시 클러스터에 걸쳐 유지된다.
  - 아래 메서드 호출 시 xxx 캐시를 먼저 확인해서 데이터가 있다면 xxxRepository.findAll()을 실행하지 않고(DB에 접근하지 않고) 캐시에 있는 데이터를 바로 반환한다.

```kotlin
@Cacheable(cacheNames = ["xxx"])
fun findAllXxx(): List<Xxx> {
    return xxxRepository.findAll()
}
```

## 1차 캐시

- **JPA나 그 구현체에서 사용되는 1차 캐시는 트랜잭션 수준에서 유지되는 캐시**이며 JPA의 persistence context(EntityManager 또는 Session)가 1차 캐시로 동작한다.
- **JPA의 persistence context 자체가 1차 캐시이므로 별도의 캐시 저장소 구성 없이도 1차 캐시는 기본적으로 활성화** 돼 있다.
- 개별 트랜잭션 수준에서만 유지되므로 일반적으로 성능 개선 효과가 매우 크지는 않다.

```kotlin
// Tx 시작

// ...

entityManager.find(Xxx:class.java, 111L)  // 111L에 대한 최초 조회 시 DB에 접근해서 1차 캐시에 저장한다.

// ...

entityManager.find(Xxx:class.java, 111L)  // 여기에서는 DB에 접근하지 않고 1차 캐시에 있는 데이터를 바로 반환한다.

// ...

// Tx 종료
```

## 2차 캐시

- **2차 캐시는 통상적인 캐시와 마찬가지로 트랜잭션 수준을 넘어서 유지**되므로, **1차 캐시의 단점을 극복하고 큰 폭의 성능 개선 효과**를 기대할 수 있다.
  - 2차 캐시로 인메모리 캐시 사용 시 JVM 수준(구체적으로는 SessionFactory 수준), 분산 캐시 사용 시 분산 캐시 클러스터 수준에서 유지된다.
- 통상적인 캐시와 마찬가지로 **캐시 저장소 구성이 필요**하며, **이미 통상적인 캐시를 위한 저장소가 구성돼 있다면 2차 캐시도 해당 저장소를 재활용할 수 있다.**
- **통상적인 캐시와 가장 큰 차이점은 `@Cacheable`로 지정한 메서드를 통해서가 아니라 JPA를 통해 데이터에 접근할 때 캐시를 활용한다는 점**이다.

```
(xxx#111 조회 요청) -> (1차 캐시) -- 없으면 --> (2차 캐시) -- 없으면 --> (DB)
```

### 2차 캐시 구성 및 사용 방법

- 검색하면 많이 나옴
  - 하이버네이트 문서: https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#caching
  - 스프링 부트 3, 하이버네이트 6 설정: https://howtodoinjava.com/hibernate/configuring-ehcache-3-with-hibernate-6/

### 2차 캐시 주의할 점

- 컬렉션 데이터를 캐시할 때 컬렉션의 원소가 
  - int 같은 기본 타입이거나, 원소가 기본 타입이 아니지만 `@ElementCollection`을 사용한다면 컬렉션을 이루는 객체 자체가 캐시되지만,
  - 기본 타입이 아니면서 `@OneToMany`나 `@ManyToOne`를 사용한다면 컬렉션의 원소인 객체의 식별자만 캐시된다.
  - https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#caching-collection 참고
- 그래서 다음과 같이 `@OneToMany`가 붙어 있는 컬렉션에만 `@Cache`를 붙이고 컬렉션의 원소인 Yyy에 `@Cache`를 붙이지 않으면,
  - 식별자만 2차 캐시에 저장되고, 
  - 이후 컬렉션을 가져올 때 2차 캐시에서 식별자 목록만 가져오고, **식별자에 해당하는 객체의 정보는 2차 캐시에 없으므로 항상 DB에서 1건 씩 별개의 쿼리로 조회해서 가져오므로 심각한 성능 저하가 발생할 수 있다.**

    ```kotlin
    class Xxx {
        
        // ...

        @OneToMany(mappedBy = "xxx")
        @Cache(
            region = "xxx.yyys",  // cache의 name(또는 alias)와 같은 값이어야 한다
            usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE,
        )
        val yyys: List<Yyy>
    }

    // 여기에 @Cache 가 없으면 성능 저하 발생
    class Yyy {
        // ...
    }
    ```

- 따라서 **컬렉션을 캐시할 때는 컬렉션의 원소가 되는 클래스에 반드시 `@Cache`를 붙여줘야 한다!**
- 그렇다고 컬렉션의 원소가 되는 클래스에만 `@Cache`를 붙이고, 컬렉션쪽에 `@Cache`를 붙이지 않아도 되냐하면 그건 아니다.
  - 컬렉션쪽에 `@Cache`를 붙이지 않으면, 캐시된 식별자가 없으므로 대략. `select col1, col2, ... from yyy where xxx_id = ?`와 같은 쿼리가 항상 실행되므로 캐시 효과를 볼 수 없게 된다.
    - `select col1, col2, ... from yyy where xxx_id = ?` 의 결과인 yyy 들은 각각 캐시에 저장되겠지만 컬렉션으로서 yyys를 불러올 때는 항상 쿼리가 실행되므로 캐시가 없는 것과 마찬가지다.

### TTL

- 예를 들어 컬렉션 캐시인 xxx.yyys 캐시의 TTL을 10분, 컬렉션 원소 클래스의 캐시인 yyy 캐시의 TTL을 30분으로 지정하면,
  - 10분 이내에는 항상 xxx.yyys 캐시에 있는 식별자를 사용해서, yyy 캐시에서 실제 데이터를 가져온다.
  - 10분 초과 30분 이내에는 xxx.yyys 캐시가 만료된 상태이므로 식별자 목록이 없어 DB에서 데이터를 가져오고(식별자 뿐만아니라 데이터도 가져옴), 이 때 가져온 yyy 데이터를 yyy 캐시에 저장하려고 하지만 yyy 캐시 TTL이 아직 남아있고 데이터 버전이 다르지 않아 non-writable 이므로 캐시에 저장되지 않는다(AbstractReadWriteAccess 소스 참고).

    ```
    o.h.c.s.support.AbstractReadWriteAccess  : Caching data from load [region=`yyy` (AccessType[read-write])] : key[yyy#5751] -> value[CacheEntry(yyy)]
    o.h.c.s.support.AbstractReadWriteAccess  : Checking writeability of read-write cache item [timestamp=`7010386888146944`, version=`0`] : txTimestamp=`7010386955526144`, newVersion=`0`
    o.h.c.s.support.AbstractReadWriteAccess  : Cache put-from-load [region=`AccessType[read-write]` (yyy), key=`yyy#5751`, value=`CacheEntry(yyy)`]. failed due to being non-writable
    ```

  - 30분 초과이후에는 두 캐시 모두 만료된 상태이므로 DB에서 데이터를 가져와서 캐시에 저장한다.
- 결국 **컬렉션으로 가져올 때는 컬렉션쪽 캐시의 TTL을 기준으로 DB 조회 여부가 나눠진다.**
- **컬렉션 조회 관점에서는 컬렉션 원소 클래스 쪽 캐시 TTL을 컬렉션 쪽 캐시 TTL보다 길게 지정해도 실익이 없다.**
- 다만 컬렉션 원소 클래스 캐시가 컬렉션 조회뿐 아니라 개별 인스턴스에 대한 조회에도 사용된다면 컬렉션 쪽 캐시 TTL과 무관하게 컬렉션 원소 클래스 캐시의 TTL이 그 자체로도 효과가 있다.

