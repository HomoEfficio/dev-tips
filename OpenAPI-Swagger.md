# OpenAPI and Swagger

OpenAPI는 규격, Swagger는 구현

Swagger가 먼저 나왔고 이를 토대로 OpenAPI가 나왔다.  
즉 표준 없는 상태에서 product가 먼저 나왔고, product를 바탕으로 표준 수립

이건 OpenAPI:Swagger = JPA:Hibernate 라고 할 수 있을 듯


## API First 개발 

https://youtu.be/YmQyzNF5iKg

- OpenAPI 문서 먼저 작성
- 도구를 활용해서 OpenAPI 문서에서 소스 코드 자동 생성
  - 도구를 Swagger로 할 경우, `@ApiXXX` 처럼 Swagger UI에 표시될 메타 정보를 담고 있는 애노테이션이 붙어 있는 웹 컨트롤러 인터페이스 소스 코드 자동 생성
- 자동 생성된 인터페이스 소스 코드를 구현하는 실제 코드 작성
- https://github.com/HomoEfficio/dev-tips/blob/master/api-first-dev-open-api.md 참고


## Source First 개발

- 소스 코드 먼저 작성 후, 소스 코드로부터 OpenAPI 문서 자동 생성
- 자동 생성된 OpenAPI 문서를 편집해서 메타 정보 편집
- 소스가 변경될 되면 OpenAPI 문서도 반복적으로 자동 생성하면서 업데이트 및 버전 관리
