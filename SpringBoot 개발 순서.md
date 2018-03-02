1. boot 프로젝트 생성
    - Spring Initializer
        - Web, JPA, H2, Actuator
    - localhost:8080 으로 테스트
    
1. application.yml에 local, dev, production 프로파일 작성

1. 도메인 객체 작성
    - 도메인 로직 테스트 케이스 작성
    - 도메인 객체 작성
    
1. repository 작성
    - repository 인터페이스 작성
    - Optional로 작성
    - repository 테스트 작성
      - 기본 JpaRepository 메서드 외의 메서드만
      - @DataJpaTest
      - @SpringBootTest를 붙이지 않으면 Repository외의 다른 Bean은 @Autowired 불가

1. 서비스 작성
    - 서비스 인터페이스 작성
    - 서비스 테스트 작성
      - 테스트 클래스에 @Transactional 붙여줘야 메서드 별로 저장 후 rollback 적용됨
    - 서비스 작성
      - @Transactional
      - 조회 메서드에도 @Transactional(readOnly = true)붙여줘야 JPA proxy 관련 오류 안 생김
      
1. 컨트롤러 작성
    - 테스트는 service를 mocking 하는 방식 대신 통합테스트와 유사하게 작성
    
        ```java
        @RunWith(SpringRunner.class)
        @SpringBootTest
        ```
   - https://github.com/HomoEfficio/spring-web-api-test-stubber 활용 가능
   
1. dto 작성
    - 뷰에 맞는 dto 작성
    - 뷰에 맞는 validation 애노테이션 포함
    
1. converter 작성
    - converter 테스트 작성
    - http://modelmapper.org/ 활용 가능
    
1. 컨트롤러에 dto 적용
    - 컨트롤러 메서드에 사용되던 도메인 객체를 dto로 변경
    
1. 서비스에 dto-엔티티 매핑 적용
    - 서비스 메서드의 파라미터로 사용되던 엔티티 객체를 dto로 변환
    - dto <-> 엔티티 변환 후 repository 호출/결과 반환
    
1. 통합 테스트 실행
    - Controller 테스트

1. Swagger 적용
    - http://springboot.tistory.com/24 참고
        - SwaggerConfig Bean 추가
        - Swagger UI 추가: app_root/swagger-ui.html 에서 확인 가능
   
1. Logging 적용
    - Lombok과 @Slf4J 활용
    
1. Biz: 도메인 작성 ~ 통합 테스트 실행 까지를 계속 iteration

1. 공통/아키텍트
    - JPA Auditing 설정    
    - DateTimeFormatter 공통화    
    - Collection의 validator 공통화    
    - BaseEntity로 JPA auditing 적용
    - XSS 필터 적용
    - JSON response XSS 처리
    - Generic을 활용한 DataConverter 일반화
    - Swagger 설정 및 적용


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
