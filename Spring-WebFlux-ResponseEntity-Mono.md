# Spring WebFlux ResponseEntity<Mono<T>> vs Mono<ResponseEntity<T>>

일상적으로는 둘 중 어느 방식을 사용하더라도 동작은 하는데,  
Spring Security `@PreAuthorize`가 붙으면 동작이 달라진다.

## ResponseEntity<Mono<T>>

컨트롤러 메서드 반환 타입이 `ResponseEntity<Mono<T>>`인 메서드에 `@PreAuthorize`를 붙이면,
메서드 구현부에 진입하지도 못하고 아래와 같은 에러가 발생한다.

```
java.lang.IllegalStateException: The returnType class org.springframework.http.ResponseEntity on public org.springframework.http.ResponseEntity 메서드이름(파라미터들) must return an instance of org.reactivestreams.Publisher (i.e. Mono / Flux) in order to support Reactor Context
	at org.springframework.security.access.prepost.PrePostAdviceReactiveMethodInterceptor.invoke(PrePostAdviceReactiveMethodInterceptor.java:76)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.MonoLift] :
	reactor.core.publisher.Mono.error
	org.springframework.web.reactive.result.method.InvocableHandlerMethod.lambda$invoke$0(InvocableHandlerMethod.java:156)
Error has been observed at the following site(s):
	|_    Mono.error ⇢ at org.springframework.web.reactive.result.method.InvocableHandlerMethod.lambda$invoke$0(InvocableHandlerMethod.java:156)
	|_  Mono.flatMap ⇢ at org.springframework.web.reactive.result.method.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:137)
	|_               ⇢ at org.springframework.web.reactive.result.method.annotation.RequestMappingHandlerAdapter.lambda$handle$1(RequestMappingHandlerAdapter.java:199)
	|_    Mono.defer ⇢ at org.springframework.web.reactive.result.method.annotation.RequestMappingHandlerAdapter.handle(RequestMappingHandlerAdapter.java:199)
	|_     Mono.then ⇢ at org.springframework.web.reactive.result.method.annotation.RequestMappingHandlerAdapter.handle(RequestMappingHandlerAdapter.java:199)
	|_ Mono.doOnNext ⇢ at org.springframework.web.reactive.result.method.annotation.RequestMappingHandlerAdapter.handle(RequestMappingHandlerAdapter.java:200)
	|_ Mono.doOnNext ⇢ at org.springframework.web.reactive.result.method.annotation.RequestMappingHandlerAdapter.handle(RequestMappingHandlerAdapter.java:201)
Stack trace:
		at org.springframework.security.access.prepost.PrePostAdviceReactiveMethodInterceptor.invoke(PrePostAdviceReactiveMethodInterceptor.java:76)
```
  
구구절절 말은 많고 실질적인 도움은 별로 안 되는 에러 메시지인데,  
결국 `메서드이름(파라미터들) must return an instance of org.reactivestreams.Publisher (i.e. Mono / Flux) in order to support Reactor Context` 이 부분이 중요한 것 같다.


## Mono<ResponseEntity<T>>

위 에러 메시지를 참고해서 반환 타입을 `Mono<ResponseEntity<T>>`로 바꾸면 `@PreAuthorize`가 붙어 있어도 정상 동작한다.


## 마무리

- 특별한 이유가 없다면 컨트롤러 메서드 반환 타입은 `Mono<ResponseEntity<T>>`로 하자. `ResponseEntity<Mono<T>>`로 하지 말고.
- 이유는.. 아몰랑~ 머지않아 Project Loom 나오면 백프레셔 고려 안 해도 되는 대부분의 웹앱에서는 Reactor 안 쓰게 될 테니, 리액터에 너무 많은 시간 쓰지 않기로.

