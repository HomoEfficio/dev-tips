# Spring Boot 3 에서 Elasticsearch 7 사용

- Spring Boot 3.2 에서 사용하는 Spring Data Elasticsearch 5.2.1 에서는 기본으로 Elasticsearch 8 을 지원
- Elasticsearch 8 은 응답 헤더에 X-Elastic-Product: Elasticsearch 을 포함하며, Elasticsearch Client 도 응답 헤더에 X-Elastic-Product 가 포함돼 있음을 검사
- Elasticsearch 7 은 응답 헤더에 X-Elastic-Product 을 포함하지 않음
- 따라서 Elasticsearch 7 을 사용하려면 응답 헤더에 X-Elastic-Product 추가하도록 설정 필요

  ```kotlin
  import org.apache.http.HttpResponseInterceptor
  import org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchProperties
  import org.springframework.context.annotation.Configuration
  import org.springframework.data.elasticsearch.client.ClientConfiguration
  import org.springframework.data.elasticsearch.client.elc.ElasticsearchClients
  import org.springframework.data.elasticsearch.client.elc.ElasticsearchConfiguration
  import org.springframework.data.elasticsearch.repository.config.EnableElasticsearchRepositories
  import org.springframework.data.mapping.model.FieldNamingStrategy
  import org.springframework.data.mapping.model.SnakeCaseFieldNamingStrategy
  
  @Configuration
  @EnableElasticsearchRepositories(basePackages =["repo-package-path"])
  class ElasticsearchConfig(
      private val properties: ElasticsearchProperties,
  ) : ElasticsearchConfiguration() {
  
      override fun clientConfiguration(): ClientConfiguration {
          return ClientConfiguration.builder()
              .connectedTo(*properties.uris.toTypedArray())
              .withConnectTimeout(properties.connectionTimeout)
              .withBasicAuth(properties.username, properties.password)
              .withClientConfigurer(ElasticsearchClients.ElasticsearchHttpClientConfigurationCallback.from { builder ->
                  builder
                      .addInterceptorLast(HttpResponseInterceptor { response, _ ->
                          response.addHeader("X-Elastic-Product", "Elasticsearch")  // 여기!!
                      })
              })
              .build()
      }
  
      override fun fieldNamingStrategy(): FieldNamingStrategy =
          SnakeCaseFieldNamingStrategy()
  }
  ```

- Elasticsearch 7 을 지원하는 Spring Data Elasticsearch 4.2.12 는 다음과 같은 보안 이슈가 있음
  ```
  Provides transitive vulnerable dependency maven:org.elasticsearch:elasticsearch:7.12.1
  CVE-2023-46673 7.5 Improper Handling of Exceptional Conditions vulnerability with High severity found
  CVE-2023-31419 7.5 Out-of-bounds Write vulnerability with High severity found
  CVE-2021-22144 6.5 Uncontrolled Recursion vulnerability with Medium severity found
  CVE-2021-22145 6.5 Generation of Error Message Containing Sensitive Information vulnerability with Medium severity found
  CVE-2021-22146 7.5 Exposure of Resource to Wrong Sphere vulnerability with High severity found
  CVE-2021-22147 6.5 Incorrect Permission Assignment for Critical Resource vulnerability with Medium severity found
  Results powered by Checkmarx(c)
  ```
>**Spring Data Elasticsearch 5.2.1 을 사용하고 Elasticsearch 버전 관련 이슈 발생 시 Spring Data Elasticsearch 4.2.12 로 다운그레이드**
