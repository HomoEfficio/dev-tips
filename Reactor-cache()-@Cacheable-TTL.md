# Spring Reactor cache(), @Cacheable, TTL

## Reactor cache()

스프링 리액터의 Mono와 Flux에 `cache()` 메서드가 있으니 뭔가 어렵지 않게 캐시 기능을 사용할 수 있을 것 같다.

다음과 같이 작성하면 끝~

```kotlin
@Service
class ItemService(
    private val itemRepository: ItemRepository,
) {

    fun getItem(id: String): Mono<Item> {
        return itemRepository.findById(id).cache()
    }
}

```

일 것 같지만, 안타깝게도 컨트롤러에서 위 `getItem()` 메서드를 호출할 때마다 새 Mono가 생성되고, 그 Mono가 subscribe 될 때마다 계속 DB를 조회한다.

그럼 위에 나오는 `cache()`는 대체 뭘 캐시한다는 걸까? 이럴 때는 어쩔 수 없이 공식 문서를 보게 된다.

https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#cache--

![Imgur](https://i.imgur.com/bCCoUS3.png)

안타까운 건 보면 알게 되면 좋겠는데, 공식 문서는, 특히 리액터 쪽은 아무리 그림으로 설명을 해도 영.. ㅠㅜ

암튼 `cache()`는 어딘가에 일정기긴 저장되는 개념이라기보다는 `cache()`가 반환값을 변수에 할당해두면 나중에 변수를 통해 재사용 할 수 있다.


## Spring @Cacheable

TODO


## TTL

TODO


