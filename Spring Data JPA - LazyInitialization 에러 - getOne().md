# Spring Data JPA - LazyInitialization 에러

스프링 데이터 JPA를 사용하다보면 가끔 만나는 정겨운 에러가 있다.

```
LazyInitializationException: could not initialize proxy - no session
```

요 에러는 말그대로 지연 로딩을 하려는데 이미 세션이 사라져서 지연 로딩을 못할 때 발생하는 에러다.

그래서 보통은 `@*ToMany` 관계로 되어 있어 기본 FetchType이 Lazy인 멤버를 읽어올 때 발생하며, 그냥 쉬운 해결 법으로는 `FetchType.EAGER`를 명시해서 즉시 로딩으로 읽어오게 하면 해결될 수도 있다. 

하지만, 비즈니스 요구가 아니라 단순히 LazyInitialization 에러를 피하기 위해 즉시 로딩을 선택하는 것은 좋은 해법이 아니다.

요는 지연 조회 시점까지 세션을 유지 시켜주면 되는데 **가장 쉽게는 해당 조회가 포함된 메서드에 `@Transactional`(조회일 때는 `@Transactional(readOnly = true)`)를 붙여주면 된다.**

그런데 이렇게 해도 계속 LazyInitialization 에러가 발생하는 경우를 만났다!!

나중에 알고보니 꽤 허탈했는데 원인은 `xxxRepository.findOne(id)`를 호출해야되는데 `xxxRepository.getOne(id)`를 호출했기 때문이었다.

아마도 오타로 무심결에 f 대신 g를 입력했는데 IDE가 자동 완성으로 `getOne`을 추천해주고, 습관적으로 그냥 추천해준대로 선택해서 getOne()이 묻어간 듯 하다.

IDE 자동 완성 기능에도 병폐가 있다는 사실을 알게 되었다..


