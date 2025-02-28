# Caffeine Cache 기본

## 일반적인 캐시

- 일반적으로 캐시는 갯수 기반 정책이든, 시간 기반 정책이든 만료(expire)가 되어 캐시에서 사라지고(evict),
- 그 이후에 조회 요청이 들어올 때 DB에서 조회를 해서 캐시를 채운다.
- 트래픽이 굉장히 많은 시스템이라면 캐시 만료와 캐시 채우기 사이에 DB를 조회하는 동안에도 계속 조회 요청이 들어오므로,
- 그 자체로 시스템에 매우 큰 부하를 줄 수 있고, 그동안 발생하는 DB 조회 + 캐시 덮어쓰기라는 낭비도 발생한다.
- 이 두 가지 문제는 [NHN Forward 2020](https://youtu.be/KxQbCy3M7Ik?si=Pb8OKHcPHQoop6nP)에서 잘 설명해준다.

## 카페인 캐시는 뭐가 특별한가?

- 카페인 캐시는 asynchronous refresh 개념이 있다.
  - 참고로 위 NHN 영상에서는 refresh 가 아니라 update 라고 소개한다.
- 쉽게 말해 일반적인 expire 시간 제약 외에 expire 보다 짧은 추가로 refresh 시간 제약을 두면,
- expire 제한 시간이 되기 전에 먼저 refresh 제한 시간에 도달하고,
- 이때부터 들어오는 조회 요청은 아직 expire 되지 않은 기존 캐시(카페인 캐시에서 사용하는 개념은 아니지만 편의상 main 캐시라고 하자)에서 조회해서 반환하고,
- 그동안 비동기로 DB를 조회해서 여벌의 캐시(카페인 캐시에서 사용하는 개념은 아니지만 편의상 secondary 캐시라고 하자)를 생성하고,
- secondary 캐시 생성이 완료되면 secondary 캐시가 main 캐시가 된다.
- 이렇게 하면 캐시 만료와 캐시 채우기 사이에 발생하는 캐시 부존재 시간을 제거할 수 있고 일반적인 캐시에 존재하는 두 가지 단점을 제거할 수 있다.

## example

```java
LoadingCache<MyCacheKey, List<MyValue>> myCache =
    Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .refreshAfterWrite(3, TimeUnit.MINUTES)
        .maximumSize(1_000)
        .recordStats()
        .build(this::getMyValues);

private List<MyValue> getMyValues(MyCacheKey cacheKey) {
    String xyz = cacheKey.getXyz();
    return myRepository.selectList(xyz);
}
```

위와 같이 지정하면 아래와 같이 동작한다.

- 캐시 항목이 1,000개인데 새로운 조회 요청이 들어오면 기존 캐시 항목에서 하나를 제거하고 새 항목을 추가하는 방식으로 캐시 업데이트 발생
- 1,000개에 도달하든 말든 마지막 갱신 시점 이후 `refreshAfterWrite`으로 지정한 3분이 경과된 후에 조회 요청이 들어오면 비동기로 DB를 조회해서 캐시 갱신 시작
- 캐시 갱신 진행 중 들어오는 조회 요청에 대해서는 아직 만료되지 않은 캐시 정보로 반환
- 캐시 갱신 완료되면 갱신된 캐시로 서비스
- 조회 요청이 끊임 없이 계속 된다면 계속 3분(`refreshAfterWrite`으로 지정) 주기로 위 과정 반복
- 조회 요청이 없는 상태로 `expireAfterWrite`으로 지정한 5분이 경과하면 캐시 evict
- 그 이후에 조회 요청이 들어오면 일반적인 캐시처럼
  - DB 조회 + 캐시 생성이 진행되며,
  - 캐시 생성 완료 전에 조회 요청이 많다면 그만큼 DB 조회 + 캐시 덮어쓰기가 반복 발생
- 결국 꾸준하게 조회 요청이 있다면 `refreshAfterWrite`로 지정한 주기로 캐시가 비동기로 갱신되며, 캐시 공백 없이 seemless 하게 서비스 가능


## 마무리

>카페인 캐시를 사용하면,  
>`refreshAfterWrite`로 지정한 주기로 캐시가 비동기로 갱신되며, 캐시 공백 없는 seamless 캐시 데이터 서비스를 손쉽게 제공할 수 있다.


## 기타

- 카페인 캐시도 `@Cacheable` 같은 스프링 캐시 애너테이션을 사용할 수 있지만,
  - 위와 같이 `LoadingCache`를 사용해서 코딩하면, `@Cacheable`의 단점인 self invocation 시 캐시 기능이 동작하지 않는 단점을 피할 수 있다.
- nginx 도 refresh 와 비슷하게 비동기 캐시 업데이트를 지원한다: [proxy_cache_background_update on](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_background_update)
