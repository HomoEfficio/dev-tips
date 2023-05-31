# Spring-Reactor-Webflux-Custom-Validator-Restrictions

## External Service Invocation

스프링 리액터 웹플럭스 환경에서는 커스텀 밸리데이터에서 외부 서비스를 호출할 수 없다.

그래서 부적절한 문구가 포함돼 있을 때 필터링 해주는 서드파티 서비스를 커스텀 밸리데이터 방식으로는 활용할 수 없다.

왜냐하면 ConstraintValidator.isValid(..)는 Boolean을 반환게 돼 있고,  
텍스트 필터를 사용하려면 isValid() 안에서 외부 서비스를 호출해야 하는데,  
외부 서비스를 호출할 때 사용되는 WebClient는 Mono<..>를 반환한다.

결국 isValid() 안에서 블로킹 없이 Mono<..> 안의 값을 읽어서 판별 후 Boolean을 반환해야 하는데,  
블로킹 없이 Boolean 반환하는 방법이 없어 보인다.

참고: https://stackoverflow.com/questions/51300425/how-to-code-custom-validator-on-webflux-that-uses-a-reactive-datasource
