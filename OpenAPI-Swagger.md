# OpenAPI and Swagger

OpenAPI는 규격, Swagger는 구현

Swagger가 먼저 나왔고 이를 토대로 OpenAPI가 나왔다.  
즉 표준 없는 상태에서 product가 먼저 나왔고, product를 바탕으로 표준 수립

이건 OpenAPI:Swagger = JPA:Hibernate 라고 할 수 있을 듯

다양한 도구가 나와 있으며, 크게 Source First 개발, OpenAPI First 개발 방식으로 진행할 수 있다.

---

# Source First 개발

- 소스 코드 먼저 작성 후, 소스 코드로부터 OpenAPI 문서 자동 생성
- OpenAPI 문서가 소스로부터 자동 생성되므로 API 나 파라미터 설명 등의 메타 정보도 자동 생성되게 하려면 소스 코드(Web Controller)에 `@ApiXXX`와 같은 애노테이션 작성 필요
  - Web Controller 를 일반 클래스로 만들지 말고, 인터페이스를 하나 만들고 인터페이스에 `@ApiXXX` 애노테이션을 추가하고 이를 구현하는 Web Controller 클래스에는 `@ApiXXX` 애노테이션을 붙이지 않으면, `@ApiXXX` 애노테이션과 주요 소스 코드를 분리할 수 있다.
- 규격이 유동적인 상황에서 주요 로직 개발을 먼저 진행해야 할 때 적합(사실 대부분이 이렇지 않을까..)

## 주요 흐름

스프링 부트를 활용한다는 가정 하에 주요 흐름은 다음과 같다

- 웹 컨트롤러를 포함하는 애플리케이션 개발
- springdoc 의존 관계 추가
- 애플리케이션 실행하면 소스 코드 기준으로 openapi 문서 생성
- 특정 URL 에서 openapi 문서 다운로드 가능
- (프론트) openapi 를 지원하는 Mock 기동 및 개발에 활용

## springdoc 의존 관계 추가

https://github.com/springdoc/springdoc-openapi

### Spring MVC

```groovy
dependencies {
    ...
    implementation 'org.springdoc:springdoc-openapi-ui:버전'
    implementation 'org.springdoc:springdoc-openapi-security:버전'  // Spring Security 가 적용돼 있을 때만 필요
    ...

```

### Spring WebFlux

```groovy
dependencies {
    ...
    implementation 'org.springdoc:springdoc-openapi-webflux-ui:버전'
    // WebFlux 용 Spring Security 도 필요 시 추가
    ...
```

### Kotlin

위 내용에 kotlin 관련 의존 관계 추가

```groovy
    ...
    implementation("org.springdoc:springdoc-openapi-kotlin:버전")
    ...

```

## 애플리케이션 실행 및 Swagger UI 확인

애플리케이션을 구동하고 http://localhost:8080/swagger-ui.html 에 접속하면 현재 개발 내용 기준으로 생성된 Swagger UI가 표시된다.


## openapi 파일 다운로드

JSON 파일은 /v3/api-docs 에 접속, YAML 파일은 /v3/api-docs.yaml 에 접속하면 다운로드 할 수 있으며, 필요 시 이 파일로 프론트와 협업한다.

## (프론트) openapi 지원 Mock 서버 기동

아래 Mock Server 내용 참고

---

# API First 개발

https://youtu.be/YmQyzNF5iKg

- OpenAPI 문서 먼저 작성
- 도구를 활용해서 OpenAPI 문서에서 소스 코드 자동 생성
  - 도구를 Swagger로 할 경우, `@ApiXXX` 처럼 Swagger UI에 표시될 메타 정보를 담고 있는 애노테이션이 붙어 있는 웹 컨트롤러 인터페이스 소스 코드 자동 생성 가능(이긴 한데 2021-06 현재 Kotlin은 아직 안 되는 듯 ㅠㅜ)
- 자동 생성된 인터페이스 소스 코드를 구현하는 실제 Web Controller 코드 작성
  - `@ApiXXX` 애노테이션을 인터페이스에 몰아 넣어 실제 Web Controller 구현 클래스와 분리 가능

## 주요 흐름

- openapi 규격을 따르는 yaml 파일 작성
- codegenerator로 yaml -> controller interface, dto 파일 자동 생성
- 프론트엔드는 prism 등 openapi를 지원하는 여러 mock 를 사용해서 개발 가능


## openapi yaml 작성

### 문법

- 문법을 숙지하면 좋겠지만 **문법을 몰라도 샘플 yaml을 바탕으로 그럭저럭 작성 가능**
  - 샘플: https://github.com/OAI/OpenAPI-Specification/tree/main/examples/v3.0

### 검증

- CLI 도구도 있고, https://editor.swagger.io/ 화면 왼쪽 에디터에 yaml 내용 붙여 넣으면 쉽게 검증 및 Swagger UI 확인 가능

### 파일 구조화 및 번들

- openapi yaml을 한 파일에 작성하다보면 분량이 굉장히 많아짐
  - 분량은 많지만 현실적으로 복붙이 많아서 실제 타이핑량은 그리 많지 않고 할만함
- **`$ref`를 적절히 사용해서 파일 분리 가능**
  - 분리 방법: https://davidgarcia.dev/posts/how-to-split-open-api-spec-into-multiple-files/
- **분리해서 작성한 파일을 `swagger-cli` 도구를 통해 하나의 번들 파일로 합칠 수 있음**
  - `swagger-cli -d bundle SOURCE-OPEN-API.yaml --outfile BUNDLE-OPEN-API.yaml --type yaml`


## yaml -> 소스 코드 생성

### openapi-generator

- [openapi-generator](https://github.com/OpenAPITools/openapi-generator)
  - CLI, jar, maven/gradle plugin, online 등 다양한 방식으로 소스 생성 가능
  - controller, model 파일 등 프로젝트 한 통이 자동 생성됨
  - openapi-generator 이므로 Swagger Annotation을 만들어주는 기능은 없음

### swagger-generator

- [swagger-codegen](https://github.com/swagger-api/swagger-codegen)
  - openapi-generator 와 거의 비슷하게 프로젝트 한 통을 자동 생성 가능
  - **swagger-codegen 이므로 스웨거 애노테이션 자동 생성 가능**
    - 다만 **자바만 지원**
    - CLI로 아래 명령 실행해보면 kotlin에서는 지원하지 않음을 확인 가능
      - swagger-codegen config-help -l kotlin-server
      - swagger-codegen config-help -l spring
        - interface 관련 옵션 있음
  - gradle plugin은 설치/설정이 maven에 비해 복잡
    - gradle plugin: https://github.com/int128/gradle-swagger-generator-plugin
    - 설치. 및 설정
      - groovy: https://github.com/int128/gradle-swagger-generator-plugin/blob/master/acceptance-test/examples/openapi-v3/java-spring/build.gradle
      - kotlin: https://stackoverflow.com/questions/54233751/trying-to-include-swagger-codegen-with-gradle-kotlindsl

          ```kotlin
          // build.gradle.kts

          plugins {
              id("org.hidetake.swagger.generator") version "2.18.2"
          }

          repositories {
            mavenCentral()
          }

          dependencies {
              implementation("io.swagger.core.v3:swagger-annotations:2.1.9")
              implementation("io.springfox:springfox-swagger2:2.10.5")
              swaggerCodegen("org.openapitools:openapi-generator-cli:3.3.4")
          }


          swaggerSources {
              create("AnyName").apply {
                  setInputFile(file("$rootDir/build/open-api-bundled.yaml"))
                  code(closureOf<org.hidetake.gradle.swagger.generator.GenerateSwaggerCode> {
                      language = "kotlin-spring"
                      components = listOf("models", "apis", "supportingFiles")
                      configFile = file("$rootDir/config.json")
                  })
              }
          }
          ```
        - 위와 같이 설정하면 gradle > Tasks > build 아래에 `generateSwaggerCodeAnyName`이 생기며 이를 실행하면 소스 코드 생성됨
        - 다만 앞서 언급한 것처럼 Swagger Annotation이 자동으로 붙어 있는 코틀린 인터페이스 파일은 생성되지 않음

# Mock Server

### prism

- [prism](https://meta.stoplight.io/docs/prism/README.md)
  - 아래와 같이 openapi yaml 파일 위치만 지정해주면 로컬에서 실행되는 mock 서버 쉽게 구동 가능
    - `prism mock ~/tmp/open-api-bundled.yaml`
  - yaml 파일만 프론트엔드와 공유하면 간단하게 contract testing 을 통한 개발 가능




