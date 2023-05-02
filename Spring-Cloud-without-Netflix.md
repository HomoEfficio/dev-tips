# Spring Cloud without Netflix

Spring Cloud 프로젝트의 시발점이기도 했던 Netflix 라이브러리가 2018.12.18부터 더 이상 기능 추가는 없고 버그 픽스만 진행되는 [Maintenance 모드에 돌입](https://spring.io/blog/2018/12/12/spring-cloud-greenwich-rc1-available-now)했다.

[Pivotal의 자료](https://www.slideshare.net/Pivotal/how-to-live-in-a-postspringcloudnetflix-world-olga-maciaszeksharma-jakub-pilimon-140430915)에 따르면 구체적인 목록은 다음과 같다.

- spring-cloud-netflix-archaius -> spring-cloud-config-server
- spring-cloud-netflix-hystrix-contract
- spring-cloud-netflix-hystrix-dashboard
- spring-cloud-netflix-hystrix-stream
- spring-cloud-netflix-hystrix -> spring-cloud-circuitbreaker + resilience4j
- spring-cloud-netflix-ribbon -> spring-cloud-loadbalancer
- spring-cloud-netflix-turbine-stream
- spring-cloud-netflix-turbine -> micrometer + prometheus
- spring-cloud-netflix-zuul -> spring-cloud-gateway

Netflix 라이브러리의 후속 버전 설정을 짧게 정리해본다.

## spring-cloud-netflix-zuul -> spring-cloud-gateway

[문서](https://cloud.spring.io/spring-cloud-gateway/reference/html/)에 따르면 Spring Cloud Gateway는 GatewayFilter를 통해 Circuit Breaker 등 기능을 제공한다. ~~그런데 2019-10-25 현재 Circuit Breaker로 [Hystrix GatewayFilter](https://cloud.spring.io/spring-cloud-gateway/reference/html/#hystrix) 밖에 구현되어 있지 않아서 Resilience4J는 아직 Spring Cloud Gateway에 사용할 수 없다.~~
update: 2020-04-29 현재 Circuit Breaker 추상화 라이브러리인 [Spring Cloud Circuit Breaker 를 통해 Resilience4j 도 사용할 수 있다](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.0.RC2/reference/html/#spring-cloud-circuitbreaker-filter-factory). 

다만 2.x 버전은 아직 M1 단계이고 샘플은 [여기](https://piotrminkowski.com/2019/12/11/circuit-breaking-in-spring-cloud-gateway-with-resilience4j/)에, 1.x 버전 샘플은 [Baeldung 사이트](https://www.baeldung.com/spring-cloud-circuit-breaker)에 있다.


### build.gradle

```groovy
  // zuul 관련 삭제
  implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
```

### application.yml

```yml
spring.cloud.gateway:
  discovery:
    locator.enabled: true
  routes:
    - id: upstream-multiplication-multiplications
      uri: lb://multiplication  # outbound url path to Eureka Server (lb://SPRING.APPLICATION.NAME)
      predicates:
        - Path=/multiplications/**  # inbound url path from UI
      filters:
        - RewritePath=/multiplication/(?<path>.*), /$\{path}  # outbound url path to spring.application.name)
    - id: upstream-multiplication-results
      uri: lb://multiplication
      predicates:
        - Path=/results/**
      filters:
        - RewritePath=/multiplication/(?<path>.*), /$\{path}
    - id: upstream-gamification-leaders
      uri: lb://gamification
      predicates:
        - Path=/leaders/**
      filters:
        - RewritePath=/gamification/(?<path>.*), /$\{path}
    - id: upstream-gamification-stats
      uri: lb://gamification
      predicates:
        - Path=/stats/**
      filters:
        - RewritePath=/gamification/(?<path>.*), /$\{path}
```

## spring-cloud-netflix-ribbon -> spring-cloud-loadbalancer

현실적으로 Ribbon을 대체하기는 이르다. 이유는 Spring Cloud Gateway는 2019-10-25 현재 [Circuit Breaker 구현체로 Hystrix 만을 지원](https://spring.io/projects/spring-cloud-gateway)하는데, **Hystrix 가 내부적으로 Ribbon에 의존**하고 있기 때문이다.

즉, **Spring Cloud Gateway를 사용하면서 Ribbon을 제거하면 Spring Cloud Gateway에 Circuit Breaker를 적용할 수 없게 된다.**

### build.gradle

```groovy
	implementation('org.springframework.cloud:spring-cloud-starter-netflix-eureka-client') {
		exclude group: 'org.springframework.cloud', module: 'spring-cloud-starter-netflix-archaius'
		exclude group: 'org.springframework.cloud', module: 'spring-cloud-starter-netflix-ribbon'
		exclude group: 'com.netflix.ribbon', module: 'ribbon-eureka'
	}
	implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
```

### application.yml

```yml
spring.cloud.loadbalancer:
  ribbon:
    enabled: false
```

### annotation

```java
// @LoadBalancerClient 나 @LoadBalancerClients로 커스터마이징 설정 지정 가능
// 안 붙이면 Round-Robin 방식의 기본 LoadBalancer가 동작함
@EnableDiscoveryClient  // <- 기존: @EnableEurekaClient
@SpringBootApplication
public class GatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(GatewayApplication.class, args);
	}

}

```
