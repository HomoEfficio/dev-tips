# Spring Test ApplicationContext Reuse를 통한 테스트 실행 속도 개선

- 생성에 적지 않은 비용이 드는 ApplicationContext를 재사용 할 수 있으면 전체적인 테스트 실행 속도 개선 가능

## TLDR

- 정확하지는 않지만 직관적인 이해를 위해 아주 짧게 요약하면
  >1. 소스 코드와 콘솔 로그를 보고, 테스트 클래스에 애너테이션으로 설정한 값과 사용하는 @MockBean, @SpyBean이 비슷한 테스트 클래스들(A, B, C라고 하자)을 선별해서
  >1. A, B, C 설정의 합집합을 추상클래스 Z에 애너테이션으로 붙이고 A, B, C의 테스트 관련 애너테이션과 @MockBean, @SpyBean을 제거하고 Z를 상속받게 만들면,
  >1. 결과적으로 A, B, C는 동일한 ApplicationContext를 사용하게 되어 ApplicationContext를 3개 만들지 않고 1개를 만들어 재사용 할 수 있으므로 테스트 실행 속도 개선 가능


## 기본 개념

- 기본적으로 개별 테스트 클래스마다 각각 ApplicationContext를 만들어서 사용
- **ApplicationContext 생성 비용이 크므로 내용적으로 동일한 ApplicationContext를 반복적으로 재생성하지 말고 캐시에 저장해두고 재사용할 수 있으면 테스트 실행 속도 개선 가능**
- Spring Test에서 제공하는 테스트 컨텍스트 캐시는 테스트에 사용되는 ApplicationContext를 캐시하며 테스트 스위트가 실행되는 JVM의 static 영역에 존재
  - **다른 테스트 클래스에서 내용적으로 동일한 ApplicationContext를 사용하면 새로 생성하지 않고 캐시에서 읽어온 ApplicationContext를 재사용**
- 여기에서 말하는 테스트 스위트(Test Suite)란 동일한 JVM 프로세스에서 실행되는 테스트 모음을 뜻함
  - 예를 들어 IntelliJ > Gradle > A submodule > Tasks > verification > test 를 실행할 때 ATest, BTest, CTest 가 모두 실행된다면, ATest, BTest, CTest는 모두 하나의 테스트 스위트에서 실행되는 테스트임
- 따라서 **동일한 테스트 스위트에서 실행되면서 내용적으로 동일한 ApplicationContext를 사용되는 테스트 클래스가 많을 수록 ApplicationContext 생성 횟수가 줄어들게 되므로 테스트 실행 속도 개선 가능**
  - 즉 **테스트 목적은 달성하면서도 동일한 ApplicationContext가 사용되도록 기존 테스트 클래스를 변경해서 테스트 실행 속도 개선 가능**


## ApplicationContext 최적화 방법

### 1. 테스트 스위트에 사용되는 ApplicationContext 캐시 현황 확인

- 테스트 클래스별 실행 콘솔 로그에서 `Spring test ApplicationContext cache statistics`로 검색하면 아래와 같이 테스트 컨텍스트 캐시 현황을 확인할 수 있음 

  >2023-03-16 21:11:30.218 DEBUG [,,] 7403 --- [    Test worker] org.springframework.test.context.cache   : Spring test ApplicationContext cache statistics: [DefaultContextCache@262d6623 size = 8, maxSize = 32, parentContextCount = 0, hitCount = 1405, missCount = 8]
  >- 현재 8개의 ApplicationContext가 생성되어 1405개의 테스트 케이스에 사용되고 있음
  >- 최대 32개까지만 캐시하며 32개를 초과하면 LRU에 의해 제거 후 새로 추가
  >- `spring.test.context.cache.maxSize` 프로퍼티로 32보다 큰 값 설정 가능

- 혹시 위 로그가 안 나오면 yml에서 아래 로깅 레벨 추가
  - `org.springframework.test.context.cache: DEBUG`

### 2. ApplicationContext를 새로 생성해서 테스트 컨텍스트 캐시에 저장하는 부분 확인

- 테스트 클래스별 실행 콘솔 로그에서 `Storing ApplicationContext`로 검색하면 아래와 같이 테스트 컨텍스트 캐시에 저장하는 것을 확인할 수 있음

  >2023-03-16 21:11:30.218 DEBUG [,,] 7403 --- [    Test worker] c.DefaultCacheAwareContextLoaderDelegate : Storing ApplicationContext [43130573] in cache under key [[MergedContextConfiguration@3084fc60 testClass = XxxTest, locations = '{}', classes = '{}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{prop.abc = abc, prop.xyz=xyz}', contextCustomizers = set[org.springframework.boot.test.autoconfigure.actuate.metrics.MetricsExportContextCustomizerFactory$DisableMetricExportContextCustomizer@67c0014b, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@2e6a5073, [ImportsContextCustomizer@57062c4d key = [@org.springframework.context.annotation.PropertySource(name="", factory=io.homoefficio.example.YamlPropertySourceFactory.class, ignoreResourceNotFound=false, encoding="", value={"classpath:/config-global/application.yml"}), @org.springframework.context.annotation.Import({org.springframework.boot.context.properties.EnableConfigurationPropertiesRegistrar.class}), @org.springframework.boot.context.properties.EnableConfigurationProperties({io.homoefficio.example.XxYyZz.class, io.homoefficio.example.AaBbCc.class})]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@5ef964ec, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@ed62b2f, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@7563f8c], contextLoader = 'org.springframework.test.context.support.DelegatingSmartContextLoader', parent = [null]]]
  - 위 내용에 나오는 locations, classes, contextInitializerClasses, ... 등의 정보가 동일한 ApplicationContext가 컨텍스트 캐시에 이미 있으면 새로 ApplicationContext를 만들지 않고 캐시에 있는 ApplicationContext를 재사용하게 됨
    - MergedContextConfiguration@xxx, testClass 값이 다르더라도 동일한 컨텍스트로 인식됨
    - ImportsContextCustomizer@xxx 값이 다르더라도 동일한 컨텍스트로 인식됨
  - 위 내용이 동일하게 나오더라도 테스트 클래스에서 사용하는 `@DynamicPropertySource, @MockBean, @SpyBean`이 다르면 ApplicationContext가 다르다고 인식됨
  - [애플리케이션 컨텍스트 동일성 판단 기준](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#testcontext-ctx-management-caching)
  - `Storing ApplicationContext`가 검색되지 않으면 해당 테스트 클래스는 ApplicationContext를 새로 생성하지 않음
    - ApplicationContext를 사용하지 않거나(`Spring test ApplicationContext cache statistics`로도 검색 결과 없음)
    - 캐시에 있는 ApplicationContext를 읽어서 사용하거나(`Spring test ApplicationContext cache statistics` 검색 결과 있음)

### 3. 추상 클래스를 만들어서 동일한 ApplicationContext를 사용할 테스트 클래스를 Grouping

- 위 과정을 통해 동일한 ApplicationContext를 사용하도록 컨텍스트를 통합할 수 있는 대상이 식별되면,
- 추상 클래스를 생성해서 여러 개별 테스트 클래스에 있던 테스트 관련 애너테이션과 `@MockBean`, `@SpyBean` 목록의 합집합을 추상 클래스로 이동하고,
- 개별 테스트 클래스는 추상 클래스를 상속받게 하면 동일한 ApplicationContext가 사용될 수 있음
  - 개별 테스트 클래스에 애플리케이션 컨텍스트 동일성을 깨는 설정이 추가되면 해당 테스트 클래스는 ApplicationContext를 새로 생성해서 사용하게 됨

## 걸림돌

- ComponentA가 ATest와 BTest 모두에서 사용되는데, ATest에서는 ComponentA를 `@Autowired`로 가져와서 사용하고, BTest에서는 ComponentA를 `@MockBean`으로 가져와서 사용하는 상황
  - ATest, BTest가 동일한 ApplicationContext를 사용하도록 동일한 하나의 추상 클래스를 상속 받게 만들면 충돌 발생
    - ex) 추상 클래스에서 `@MockBean`으로 ComponentA를 가져오면, 기존에 `@Autowired`로 ComponentA를 가져와 사용하던 ATest에서 NPE 발생할 수 있음
  - ApplicationContext를 재사용하게 만들려면 ATest와 BTest에서 ComponentA를 가져오는 방식을 통일해야 함
    - 이런 통일이 테스트 본연의 목적에 어긋나거나 문제 없이 잘 실행되던 테스트 일부 재작성이 필요할 수도 있음
      - 테스트 본연의 목적을 일부 양보 및 일부 재작성을 감수하고 ApplicationContext 재사용을 선택할 것인지, ApplicationContext 재사용을 포기하고 테스트 본연의 목적에 충실하고 재작성을 하지 않을 것인지 trade-off가 발생하며 엔지니어링 결정 필요
