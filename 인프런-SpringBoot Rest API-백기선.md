# 백기선 SpringBoot REST API

https://www.inflearn.com/course/spring_rest-api/ 수강 중 참고할 만한 내용 요약

## HATEOAS, HAL

- 컨트롤러에서 ResponseEntity 의 body로 ResourceSupport를 상속받는 클래스를 넘겨주면 unwrap 되지 않은 상태로 반환
- ResourceSupport 대신 Resource<Event> 를 상속받는 클래스를 넘겨주면 unwrap 된 상태로 반환한다.

    ```java
    ControllerLinkBuilder selfLinkBuilder = linkTo(EventController.class).slash()
    URI createdUri = selfLinkBuilder.toUri();
    EventResource eventResource = new EventResource(event);
    eventResource.add(linkTo(EventController.class).withRel("query-events"));
    eventResource.add(selfLinkBuilder.withSelfRel());
    eventResource.add(selfLinkBuilder.withRel("update-event"));
    return ResponseEntity.created(createdUri).body(eventResource);
    ```

## Spring REST Docs

- MockMvc, REST Assure 등을 사용하는 Controller 테스트 클래스에 `andDo(document(...))`를 통해 문서화

    ```java
    @TestConfiguration
    public class RestDocsConfiguration {
      
        @Bean
        public RestDocsMockMvcConfigurationCustomizer restDocsMockMvcConfigurationCustomizer() {
            return new RestDocsMockMvcConfigurationCustomizer() {
                @Override
                public void customize(MockMvcRestDocumentationConfigurer configurer) {
                    // JSON formatting 기능 추가
                    configurer.operationPreprocessors()
                      .withRequestDefaults(prettyPrint())
                      .withResponseDefaults(prettyPrint());
                }
            };
        }
    }
    ```

    ```java
    @AutoConfigureRestDocs
    @Import(RestDocsConfiguration.class)
    public class EventControllerTests {
      
        mockMvc.perform(...)
            ...
            .andDo(document("create-event"));  // create-event 폴더 아래에 adoc 파일 자동 생성
        
    }
    ```

- adoc를 문서로 빌드

    - asciidoc maven plugin 설치(설정도 포함)
    - index.adoc 파일 작성(레퍼런스 문서에서 복사)
    - Maven > pacakge 실행 -> generated-docs/index.html 생성 -> 서버에서 /docs/index.html 로 접근 가능



## 컨트롤러 테스트

### 슬라이싱 테스트

- `@WebMvcTest` + `@MockBean` + Mockito 사용
- Mock DispatcherSevlet을 통해 Request/Response 처리
- 슬라이싱 테스트라서 서비스나 리포지토리 관련 빈은 주입해주지 않음
    - 따라서 `@MockBean`과 Mockito를 써서 서비스나 리포지토리의 동작을 Stubbing 해야한다.

- 코드 쪼가리

    ```java
    // value() 안에 Matcher 사용 가능
    MockMvc.perform(...)
        .andExpect(jsonPath("id").value(Matchers.not(100)))
        ...
    ```

### 약식 통합 테스트

- `@SpringBootTest` + `@AutoConfigureMockMvc`
- Mock DispatcherServlet을 사용하지만 서비스나 리포지토리는 실제 Bean을 주입 받아 사용 가능
- 귀찮은 Mocking, Stubbing을 피할 수 있는 장점이 있지만 실제 DB에까지 저장하므로 수행 시간은 오래 걸릴 수 있음

### Validation

- JSR-303은 데이터 항목 하나에 대한 형식적 유효성(empty, max, min 등) 검증을 담당
- 여러 데이터 항목에 걸친 유효성 검증(시작 날짜와 종료 날짜 등) 등은 별도의 Custom Validator를 만들어야 함
- 저장된 데이터와 교차 검증이 필요한 경우(이메일 중복 여부 등)에는 서비스에서 검증해야 함


### 기타

#### DTO에 없는 속성이 들어올 때 Bad Request 처리

```
# application.yml

spring.jackson.deserialization.fail-on-unknown-properties: true
```

#### 테스트 설명용 Custom Annotation

- 테스트 이름에 테스트의 의도/동작 등을 모두 담기 어려우므로 커스텀 애노테이션 사용

  ```java
  @Target(ElementType.METHOD)
  @Retention(RetentionPolicy.SOURCE)
  public @interface TestDescription {   
    String value();
  }
  ```

- JUnit5 에는 테스트 설명 애노테이션이 제공됨

#### `@JsonComponent`

- Custom Serializer 클래스에 `@JsonComponent`를 붙여주면 Bean으로 등록되는 ObjectMapper에 Custom Serializer가 자동으로 등록된다.


#### Parameterized Test

- Dependency 추가: JUnitParams
- 코드

    ```java
    @RunWith(JUnitParamsRunner.class)
    public class XXXTest {

      @Test
      @Parameters(method = "paramsForTestFree")
      public void testFree(int basePrice, int maxPrice, boolean isFree) {

      }

      private Object[] paramsForTestFree() {
          return new Object[] {
              new Object[] { 0, 0, true },
              new Object[] { 100, 0, false },
              new Object[] { 0, 100, false },
              new Object[] { 100, 200, false }
          };
      }

    }    
    ```

#### Hibernate 설정

```
spring.jpa:
  hibernate.ddl-auto: create-drop
  properties.hibernate:
    jdbc.lob.non_contextual_creation: true
    format_sql: true

logging.level.org.hibernate:
  SQL: DEBUG
  type.descriptor.sql.BasicBinder: TRACE
```

#### docker로 postgresql 실행

1. docker 컨테이너 실행

```
docker run --name rest-api -p 5432:5432 -e POSTGRES_PASSWORD=pass -d postgres
```
- `--name`: 컨테이너 이름
- `-p`: 컨테이너:로컬 포트 매핑
- `-e`: 환경 변수
- `-d`: 데이터베이스 이름

1. docker 컨테이너 bash

```
docker exec -i -t rest-api bash
```
- `-i`: 인터랙티브 모드
- `-t`: 타겟 컨테이너
- `bash`: 컨테이너 안에서 bash 실행

1. docker 컨테이너 안에서 postresql 실행

```
su - posttres
psql -d postres -U postgres
```

1. postgresql 콘솔

- `\l`: 데이터베이스 목록 확인
- `\dt`: 테이블 목록 확인




#### 테스트에 프로파일 적용

- test/resoruces 폴더에 application-test.yml 파일 생성
- override 할 설정 내용만 application-test.yml 파일에 작성    
- 테스트 클래스에 `@ActiveProfiles("test")`를 붙여주면
    - application-test.yml 설정 내용이 적용되고,
    - 나머지 항목은 기존의 application.yml 파일 내용이 테스트에도 그대로 적용됨
