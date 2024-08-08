# Aggregate Root를 통한 엔티티 접근 이슈

```kotlin
class AggregateRoot(
    var id: String,

    @OneToMany(mappedBy = "parent")
    val childEntities: List<Child> = emptyList(),
) {
    fun getChild(childId: String): Child =
        childEntities.find { it.id == childId }
}

class Child(
    var id: String,
    
    @ManyToOne
    @JoinColumn(name = "aggregate_a_id"
    val parent: AggregateRootA,
)
```

애그리것루트를 통해서만 child 에 접근하도록 구현하면 결국 'aggregateRoot.getChild(childId)' 같은 방식으로  
리스트의 모든 원소를 가져온 후에 식별자를 비교해서 특정 child를 가져올텐데,  
이렇게 하면 결국 필요하지 않은 나머지 childEntities 모두를 가져와야 하므로 비효율적이다.

특히 childEntities와 Child에 2차 캐시를 적용하면,  
2차 캐시에는 childEntities가 저장되는 게 아니라 childEntities의 각 childEntity의 식별자들만 저장하며,  
childEntiites 조회 시 캐시된 식별자 목록을 가져온 후 각 식별자에 해당하는 childEntity 하나하나를 캐시에서 가져온다.

쉽게 말해 childEntities가 1000개라면 캐시를 1000번 호출하므로,  
원격 캐시인 경우 1000개를 한 번에 조회하는 DB 호출보다 오히려 더 오래 걸리게 된다.  
그래서 결국에는 서비스 계층에서 애그리것루트를 통하지 않고 직접 childRepository를 통해서 필요한 child만 DB를 통해 가져오도록 구현하게 된다.

이렇게 보면 '애그리것 루트를 통해서만 엔티티에 접근하게 한다'는 것은 개념적으로는 그럴 듯하지만,  
실제로는 상황에 따라 굉장히 큰 성능 손실을 유발하므로 현실적으로는 지킬 수 없는 원칙 아닌가?
