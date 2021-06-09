# OpenAPI 를 사용한 API first 개발

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

## mock server

### prism

- [prism](https://meta.stoplight.io/docs/prism/README.md)
  - openapi yaml 파일만 지정해주면 로컬에서 실행되는 mock 서버 쉽게 구동 가능
    - `prism mock ~/gitRepo/zepeto/creator-studio/studio-world-creator/build/open-api-bundled.yaml `
  - yaml 파일만 프론트엔드와 공유하면 간단하게 contract testing 을 통한 개발 가능
