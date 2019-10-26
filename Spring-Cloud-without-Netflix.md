# Spring Cloud without Netflix

Spring Cloud 프로젝트의 시발점이기도 했던 Netflix 라이브러리가 [2018.12.18부터](https://spring.io/blog/2018/12/12/spring-cloud-greenwich-rc1-available-now)으로 더 이상 기능 추가는 없고 버그 픽스만 진행되는 Maintenance 모드에 돌입했다.

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
