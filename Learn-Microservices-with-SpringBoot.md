# Learn Microservices with SpringBoot

## TDD 개발 과정

- 도메인 객체 작성
  - 일단 Persistence 적용 안 한 상태로(즉, JPA 적용 없이)
    - 애노테이션을 붙여가면서 실패한 테스트 통과
- 서비스 인터페이스 작성
- 서비스 테스트 작성
  - 서비스 객체 생성 시 필요한 의존 관계는
    - @SpringBootTest라면 @MockBean 이나 @Autowired로
    - 아니라면 @Mock + @BeforeEach Mockito.initMock(this)
- 테스트를 만족하는 서비스 구현체 작성
- 서비스 인터페이스에 메서드 추가
- 서비스 테스트 작성
- 테스트를 만족하는 서비스 구현체 작성
- 반복/리팩터링
- 더미 컨트롤러 작성
- 컨트롤러 테스트 작성
  - @WebMvcTest: Controller 관련 context만 포함해서 스프링부트 애플리케이션 실행
    - 컨트롤러가 의존하는 서비스는 @MockBean
    - @Autowired MockMvc
    - JacksonTester<T>
    - @BeforeEach JacksonTester.initFields(this, new ObjectMapper());
    - mvc.perform(get(URL).accept(MediaType)).andReturn().getResponse();
    - mvc.perform(post(URL).contentType(MediaType).content(jacksonTester.write(BODY).getJson())).andReturn().getResponse();
    - assertThat(response.getStatus()).isEqualTo(HttpStatus.OK.value());
    - assertThat(response.getContentAsString()).isEqualTo(jacksonTester.write(BODY).getJson())
- 반복/리팩터링
- UI 작성
- 도메인에 JPA 적용
  - JPA Repository 생성
  - JPA 적용 후 애노테이션 안 붙인 상태에서 `@DataJpaTest` 수행 -> 실패
- 서비스 테스트 작성
  - @Mock Repository
  - @BeforeEach Mockito.initMocks(this); Service service = new ServiceImpl(repositories...);
  - @Test given(repository.findBy...()).willReturn(Optional.empty()); verify(
  repository).save(argThat(x -> x.getY().equals(Z)));  **`<-- 특정 값을 가진 엔티티가 저장되는지 확인 가능`**
  - 아직까지 서비스 메서드 테스트에 스프링 컨텍스트 사용하지 않음
- 테스트를 만족하는 서비스 구현체 작성
  - Repository를 사용하는 코드 추가
  - U가 T를 필드로 가지고 있다고 할 때 findBy...()로 받은 optT를 U에 포함시켜서 저장할 때, `repository.save(new U(optT.orElse(인자로전달받은t))`
- [연습] 값 객체가 중복 저장되지 않게 하라
  - Test: DataJpaTest
  - Impl: entity의 equals(), hashCode()를 값으로만 비교하도록 수정
- 서비스 테스트 작성
  - stubbing 리스트 결과 그대로 리턴하는 테스트
  - Collections.emptyList()를 반환하는 dummy 구현체 작성
- 테스트를 만족하는 서비스 구현체 작성
  - Collections.emptyList() 대신 실제 조회 결과를 반환하는 구현체 작성
- 컨트롤러 테스트 작성
  - 서비스를 stubbing 하고 jacksonTester로 조회 결과 단언하는 테스트 작성
  - endpoint 가 없으므로 테스트는 405로 실패
- 컨트롤러 endpoint 작성
  - 추가된 서비스를 호출하는 endpoint 작성
- 조회 결과 표시 UI 반영

## AMQP 적용

### Pub쪽

- dependency 추가
- TopicExchange, MessageConverter, RabbitTemplate 빈 생성
- 설정 시 "" 안에 들어가는 문자열 오타 주의
- 이벤트 추가
  - **객체 참조 대신 ID만**
  - ID 외에 더 필요한 데이터는 API에 ID를 통해서 다시 조회
- EventDispatcher가 사용될 서비스에 테스트 작성
  - 단순 호출 여부만 검사하는 verify 추가
    - EventDispatcher가 Mock이라 그런지 몰라도
    - 서비스 메서드에 @Transactional이 붙어 있고
    - save() 전에 send() 호출 코드가 있고,
    - save() 시 예외 발생하더라도 verify()에서는 send()가 호출되는 것으로 나옴
- 서비스에서 eventDispatcher.send(event)로 이벤트 발송

### Sub쪽

- dependency 추가
- RabbitMQ 설정
  - TopicExchange, Queue, Binding, MessageConverter, DefaultMessageHandlerMethodFactory 빈 생성
  - 설정 시 "" 안에 들어가는 문자열 오타 주의
  - RabbitListenerEndpointRegistrar.setMessageHandlerMethodFactory(methodFactory)
  - application.yml: exchange, queue, routing-key, solved.key
- EventHandler
  - @RabbitListener(queues = "${multiplication.queue}")

## RabbitMQ 실행

- brew install rabbitmq
- 시작: rabbitmq-server start
- 종료: CTRL + C
- Mgmt UI: localhost:15672, guest/guest

## UI 분리

- Jetty

## API Gateway

- Spring Cloud Gateway(Netty, WebFlux 기반)로 구성
- 처음에는 Eureka와 연동되지 않도록 구성
  - spring.cloud.gateway.discovery.locator.enabled: false 가 디폴트라서 별도 설정하지 않으면 Eureka와 연동되지 않음
  - Upstream Host에 대한 정보를 Eureka가 아닌 API Gateway의 yml 파일에 하드코딩으로 설정
  - Routing 정보를 RouteLocator 빈을 반환하는 메서드에 포함
- 웹UI가 Gateway를 호출하도록 변경
- CORS 처리

## Service Discovery

- Eureka로 구성
- Eureka 서버 애플리케이션 새로 추가
  - build.gradle
    - spring-cloud-netflix-eureka-server
- Multiplication, Gamification, Gateway 에 Eureka 클라이언트 설정
  - build.gradle
    - spring-cloud-netflix-eureka-client
  - application.yml
    - Eureka 서버 위치 지정: eureka.client.service-url.default-zone: http://localhost:8761/eureka/
  - bootstrap.yml
    - spring.application.name: multiplication 등 (Eureka에 등록될 서비스 이름, 나중에 Gateway에서 이 값 그대로 사용)
- Gateway 에 Routing 설정 수정
  - RouteLocator 빈 제거
  - yml
      ```yml
      spring.cloud.gateway.discovery.locator.enabled: true
      spring.cloud.gateway.routes:
        - id:
          uri: lb://mutiplictation  # outbound url path to spring.application.name)
          predicates:
            - Path=/multiplications/**  # inbound url path from UI
          filters:
            - RewritePath=/multiplication/(?<path>.\*), /$\{path}
      ```

## 마이크로서비스별 멀티 인스턴스 구성

- 현재 마이크로서비스별 하나의 인스턴스로 구성
  - DB도 하나의 인스턴스에 들어있음
  - 여러 인스턴스가 공동의 DB를 사용하도록 DB 분리 필요(필요하다면 DB 클러스터링 구성)
  - 클러스터로 구성되더라도 앱 단 소스 코드에서는 단일 DB인스턴스인지 DB 클러스터인지 구분 불필요
  - 예제의 H2 는 JDBC URL에 AUTO_SERVER=TRUE; 추가하고,
  - **H2를 포함하는 애플리케이션 실행 시 `-Dh2.bindAddress=127.0.0.1`을 추가해줘야함**
- 현재 하나의 RabbitMQ 사용
  - 이미 분리돼있으므로 기본적으로는 여러 인스턴스가 붙어도 큰 변화 없음
    - 개별 Pub은 RabbitMQ에서 공유되는 큐의 worker로 동작
  - 소비한 메서드가 처리되지 못하고 Sub가 종료되면 메시지 유실 발생, 동일 메시지를 2개 이상의 Sub가 중복 소비 발생 등의 이슈에 대한 대응책 필요
  - 필요시 RabbitMQ도 클러스터 구성 가능
  - 클러스터로 구성되더라도 앱 단 소스 코드에서는 단일 MQ인스턴스인지 MQ 클러스터인지 구분 불필요

## Load Balancing

- Spring Boot 2.2.0.RELEASE + Spring Cloud Gateway Hoxton.M3 + Spring Cloud Eureka Client Hoxton.M3 를 사용해서 Spring Cloud Gateway 애플리케이션을 구동하면 다음과 같은 WARN 로그가 찍히면서, Ribbon 대신 BlockingLoadBalancerClient를 사용하라고 한다.

```
2019-10-23 23:17:49.679  WARN 66345 --- [  restartedMain] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2019-10-23 23:17:49.744  WARN 66345 --- [  restartedMain] eactorLoadBalancerClientRibbonWarnLogger : You have RibbonLoadBalancerClient on your classpath. LoadBalancerExchangeFilterFunction that uses it under the hood will be used by default. Spring Cloud Ribbon is now in maintenance mode, so we suggest switching to ReactorLoadBalancerExchangeFilterFunction instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
```

yml 파일에서 `spring.cloud.loadbalancer.ribbon.enabled`를 false로 해도 여전히 위와 같은 WARN이 나오며, build.gradle에서 다음과 같이 ribbon을 exclude해도 계속 WARN이 난다. 그래서 걍 포기

```groovy
implementation('org.springframework.cloud:spring-cloud-starter-netflix-eureka-client') {
   exclude group: 'org.springframework.cloud', module: 'spring-cloud-starter-netflix-ribbon'
}
implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
```


## Ribbon Load balancing 전략 수정

- IPing, IRule 빈을 반환하는 Configuration 클래스 추가
- 요청 전송 전에 Ping으로 target instance 가 available한지 체크한 후 요청 발송


## Circuit Breaker

- 



## Feign

- 그냥 RestTemplate 써도..

## 분산 시스템 테스팅 with Cucumber

- Cucumber 테스트를 블랙박스 테스트로 실행하기 위해 별도의 plain java 프로젝트 생성
  - Cucumber, JUnit, HttpClient, Jackson 등만 의존 관계로 포함
- \*.feature 파일 직접 작성. 자연어와 가까운 Gherkin(거킨) 언어
- feature 파일을 input으로 해서 Test Skeleton 자동 생성
- 자동 생성된 Skeleton 보완
- Skeleton 안에 테스트 코드 직접 작성
- 실행 및 확인


curl -X POST -H "Content-Type: application/json" -d '{"user": {"alias":"moises"},"multiplication":{"factorA":"42","factorB":"10"},"resultAttempt":"420"}' http://localhost:8080/results

