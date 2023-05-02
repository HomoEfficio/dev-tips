# Spring WebFlux RequestBody - Raw vs Mono

WebFlux 사용 시 Controller 단에서 `RequestBody` 를 인자로 받을 때,

다음과 같이 `Mono`를 받아오도록 작성해야할까?

```java
    @PostMapping("/mono")
    public Mono<SellerOut> createWithMono(@RequestBody Mono<SellerIn> sellerIn) {
        return sellerService.createWithMono(sellerIn);
    }
```

아니면 그냥 Raw 객체를 받아오도록 작성해야할까?

```java
    @PostMapping("/entity")
    public Mono<SellerOut> createWithRaw(@RequestBody SellerIn sellerIn) {
        return sellerService.createWithRaw(sellerIn);
    }
```

non-blocking 드라이버를 제공해주는 MongoDB를 사용한다고 가정하고 사용성과 성능 관점에서 살펴보자.

## 사용성

컨트롤러 단에서는 위에서 보는 것처럼 큰 차이가 없다. 서비스가 호출하는 데이터 접근 계층(Data Access Layer)에서 작지만 큰 차이가 발생한다.

### ReactiveCrudRepository

스프링 WebFlux 환경에서도 사용하기 쉽게 추상화 된 `ReactiveCrudRepository`를 통해 쉽게 데이터 저장소에 접근할 수 있다. MongoDB 용으로 특화된 `ReactiveMongoRepository`를 사용하면 Spring Data 에서 제공해주는 편리한 [메서드 이름 조합 방식](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#mongodb.repositories.queries)을 WebFlux에서도 사용할 수 있다. 이는 사용성과 생산성을 높여주는 데 큰 역할을 한다.

그런데 엔티티를 저장할 때 사용되는 `save()` 메서드는 다음과 같이 Mono가 아니라 Raw 객체를 사용하도록 정의돼 있다.

```java
@NoRepositoryBean
public interface ReactiveCrudRepository<T, ID> extends Repository<T, ID> {

  /**
   * Saves a given entity. Use the returned instance for further operations as the save operation might have changed the
   * entity instance completely.
   *
   * @param entity must not be {@literal null}.
   * @return {@link Mono} emitting the saved entity.
   * @throws IllegalArgumentException in case the given {@literal entity} is {@literal null}.
   */
  <S extends T> Mono<S> save(S entity);  // Mono가 아니라 걍 S
```

그래서 컨트롤러 단에서 `RequestBody`를 `Mono`로 받아오도록 만들면 `save()` 호출 전에 `block()` 같은 것을 호출해서 `Mono`에서 Raw 객체를 꺼내서 `save()`에 전달해줘야 한다. 대략 이런 식이 된다.

```java
sellerRepository.save(sellerIn.block().toEntity());
```

쓰기도 안 좋고 보기도 안 좋고 무엇보다 중간에 `block()`을 호출하므로 `Mono`를 사용하는 의미가 퇴색된다.

정리하면 결국,

>**`RequestBody`를 `Mono`로 받아오면 `ReactiveCrudRepository`랑 궁합이 별로다**

### ReactiveMongoTemplate

`ReactiveMongoTemplate`에는 다음과 같이 `Mono` 타입과 Raw 객체 모두 사용해서 저장할 수 있는 API를 제공해준다. 따라서 저장은 사용성에 큰 차이가 없다.

```java
  @Override
  public <T> Mono<T> save(Mono<? extends T> objectToSave) {
    ...
  }

  @Override
  public <T> Mono<T> save(Mono<? extends T> objectToSave, String collectionName) {
    ...
  }


  public <T> Mono<T> save(T objectToSave) {
    ...
  }
  
  public <T> Mono<T> save(T objectToSave, String collectionName) {
    ...
  }
```

하지만 `ReactiveMongoTemplate`를 사용해서 조회할 때 메서드 이름 조합 방식을 사용할 수 없으므로, 아래에 보는 것처럼 `Query`를 전달해줘야 하므로 아무래도 `ReactiveCrudRepository` 보다는 사용성이 떨어진다고 할 수 있겠다.

![Imgur](https://i.imgur.com/eogpnac.png)

>**`RequestBody`를 `Mono`로 받아오면 `ReactiveMongoTemplate`를 사용하면 되지만, 조회할 때 `ReactiveCrudRepository`보다는 불편하다**


## 성능

처음에는 예전 Servlet 관점으로 생각을 해서 `RequestBody`를 `Mono`로 받아온다 하더라도 결국은 이미 Request에서 추출해서 만들어진 Raw 객체를 `Mono`로 wrapping 해서 컨트롤러에 넣어주는 것일 거라 생각했는데 그렇지는 않은 것 같다.

WebFlux에서는 `HttpServletRequest`가 아니라 reactive 방식의 `ServerHttpRequest`를 사용한다. 따라서 Servlet 에서와는 다르게 `RequestBody`를 `Mono`로 받아오면 Raw 객체를 받아 사용하는 것과는 다르게 정말로 reactive한 처리가 가능할 수도 있겠다.

reactive 방식은 처리 속도보다는 자원 사용 관점에서의 효율성을 높이는 방식이므로 속도로 비교하는 것이 합당하지 않을 수는 있지만, 그래도 참고 삼아 k6(https://k6.io/)로 가상사용자 100 으로 10초간 3회 돌린 결과는 다음과 같다. 그림 위쪽이 Raw 방식, 아래쪽이 `Mono` 방식이다. `http_reqs` 항목으로 비교해보면 두 방식에서 의미있는 큰 차이는 없는 것으로 보인다.

![Imgur](https://i.imgur.com/R3fX3fg.png)

![Imgur](https://i.imgur.com/FDRl4jU.png)

![Imgur](https://i.imgur.com/VhCeVTk.png)


## 선Raw후Mono? 아니면 선Mono후Raw?

Request에서 먼저 Raw 가 추출된 후에 Mono로 감싸져서 컨트롤러에 전달되는 걸까? 아니면 Request에서부터 계속 Mono(또는 Flux)인 채로 있다가 나중에 Raw가 추출되서 컨트롤러에 전달되는 걸까?

Stack을 뒤져본 결과 분기 지점은 아래와 같음을 확인했다. 하지만 여기에서 위 질문에 대한 답은 얻을 수 없었다.

![Imgur](https://i.imgur.com/L6TuAEE.png)

조금 더 살펴본 결과 아래 그림 단계까지는 netty의 `NettyDataBuffer`로 존재하다가,

![Imgur](https://i.imgur.com/RNY24EO.png)

아래 그림 단계에서 처음으로 Raw 값이 확인된다.

![Imgur](https://i.imgur.com/enkERzn.png)

하지만 선Raw후Mono 인지 선Mono후Raw 인지는 아직 명확하지 않다.


## 마무리

WebFlux 사용 시 Controller 단에서 `RequestBody` 를 인자로 받을 때,

>- `Mono`를 받아오도록 작성하면 `ReactiveCrudRepository`를 사용하기 불편해지고,  
Raw 객체를 받아오도록 작성하면 `ReactiveCrudRepository`를 편하게 쓸 수 있다.
>
>- `Mono`로 받아오든 Raw 객체로 받아오든 아주 특수한 상황이 아니라면 성능 상 둘 중 어느 한 쪽이 훨씬 유리하다고 할 수는 없을 것 같다.




  
