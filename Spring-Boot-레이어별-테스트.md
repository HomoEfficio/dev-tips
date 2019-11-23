# Spring Boot 레이어 별 테스트

SpringMVC + Spring Data JPA + Spring Data Repository 를 사용한 일반적인 Spring Boot API 서버에서 레이어 별 테스트 전략을 알아보자.

## Domain

- 순수한 Domain 로직은 JUnit 나 AssertJ 등 테스트 편의 도구 외에 다른 프레임워크의 도움이 필요하지 않다.

## Repository

- 저장 및 조회와 관련된 Repository 테스트는 JPA의 도움이 필요하므로 `@DataJpaTest` 슬라이스 테스트를 활용한다.
- `@DataJpaTest` 테스트는 저장을 위한 JPA 연관 관계가 적절히 구성되었는지, Repository 메서드가 제대로 구현되었는지 확인하는 것이 목적이다.
- 먼저 JPA 관련 애노테이션 없이 코드를 작성해서 저장에 실패하는 테스트 코드를 먼저 작성하고,
- JPA 규칙에 맞는 애노테이션을 추가해서 테스트 코드를 통과시킨다.
- 테스트에 사용할 DB를 직접 지정하려면 다음 애노테이션을 추가해야 한다. 이유는 [여기](https://github.com/HomoEfficio/dev-tips/blob/master/Spring%20%40DataJpaTest%20사용%20Tips.md#embedded-db-말고-실제-db를-사용하고-싶을-때) 참고.

    ```java
    @AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
    ```
- 전형적인 Repository 테스트 코드 템플릿은 다음과 같다.

    ```java
    @DataJpaTest
    @AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
    @ActiveProfiles("test")
    class MemberRepositoryTest {

        @Autowired
        private MemberRepository memberRepository;

        @Test
        @DisplayName("새 Member 저장")
        public void create_new_member() {
            // given
            Member member = generateMember();

            // when
            Member dbMember = memberRepository.save(member);

            // then
            assertThat(dbMember.getUsername()).isEqualTo("Homo Efficio");
            assertThat(dbMember.getEmail()).isEqualTo("homo.efficio@gmail.com");
            assertThat(dbMember.getPassword()).isEqualTo("VedicMath9(");
        }

        ...

    }
    ```

## Service

- 서비스 레이어는 데이터의 CRUD를 Repository에 위임하고 트랜잭션을 관리하는 것이 주요 책임이다.
  - JPA를 사용한다면 DTO <-> Entity 간 변환도 서비스 레이어에서 처리하는 것을 선호한다. 이유는 이 변환을 컨트롤러 레이어에서 수행한다면 Lazy 조회 시 LazyInitializationException이 발생할 위험에 노출되기 때문이다.
  - 물론 이 경우에도 실제 DTO <-> Entity 간 변환 로직 자체는 DTO에 담고, 서비스 레이어에서는 DTO의 변환 로직을 호출할 뿐이다.
- Repository의 동작은 위의 Repository 테스트에서 이미 확인했으므로, 서비스 레이어에서는 Repository 메서드가 제대로 호출되는지만 확인하면 되므로 실제 Repository 대신 Mock Repository를 사용하면 된다. 따라서 Mockito 필요
    - 필요하다면 `@SpringBootTest(classes = {A.class, B.class, ...})`도 사용 가능하다.
- Mock을 사용하므로 실제 저장/조회가 발생하지 않는다.
- 생성, 삭제, 조회 시에는 생성/삭제/조회를 호출하는 메서드 호출 여부를 verify 하면 된다.
- 수정 시에는 명시적으로 `save()`나 `saveAndFlush()`가 호출되지 않을 수도 있으므로 verify 로는 테스트가 불가능하므로 Mock 에 사용되는 엔티티나 DTO의 상태 변경을 assert 한다.
- Mock에 사용되는 엔티티나 DTO를 생성하는 부분이 매우 번거로울 수 있다.
- 전형적인 서비스 레이어 테스트 코드 템플릿은 다음과 같다.

    ```java
    @ActiveProfiles("test")
    class MemberServiceTest {

        private MemberService memberService;

        @Mock
        private MemberRepository memberRepository;

        private JacksonTester<MemberDto> jsonMemberDto;

        @BeforeEach
        public void setup() {
            MockitoAnnotations.initMocks(this);
            JacksonTester.initFields(this, new ObjectMapper());
            memberService = new MemberServiceImpl(memberRepository);
        }

        @Test
        @DisplayName("새 Member 저장")
        public void create_new_member() {
            // given
            MemberDto memberDto = new MemberDto(
                    null, "Homo Efficio", "homo.efficio@gmail.com", "VedicMath9("
            );
            BDDMockito.given(memberRepository.findById(any()))
                    .willReturn(Optional.empty());
            BDDMockito.given(memberRepository.save(argThat(member -> member.getUsername().equals("Homo Efficio"))))
                    .willReturn(new Member(memberDto.getUsername(), memberDto.getEmail(), memberDto.getPassword()));

            // when
            MemberDto dbMemberDto = memberService.save(memberDto);

            // then
            BDDMockito.verify(memberRepository).save(argThat(member -> member.getUsername().equals("Homo Efficio")));
        }

        ...

    }
    ```

## Controller

- 컨트롤러는 넓게 보면 사용자가 보낸 요청을 서비스에 전달하기까지 모든 과정을 포괄한다고 볼 수 있지만, 필터, 인터셉터, 요청 라우팅, 보안(인증, 인가), 데이터 validation, 데이터 바인딩 등은 스프링 프레임워크에서 담당해주므로 컨트롤러가 실제로 담당하는 부분은 이를 모두 통과한 후의 요청 데이터를 서비스에 전달해주고, 서비스가 반환하는 결과를 클라이언트에게 반환하는 부분이다.
- 따라서 컨트롤러의 로직이 많지 않은 경우 서비스를 Mock 해서 컨트롤러 레이어만을 단위 테스트하는 것은 효용이 크지 않을 수 있으므로, 컨트롤러 테스트는 보통 Mock 대신 실제 서비스 및 도메인 계층을 대상으로 통합 테스트로 작성하는 편이 낫다.
- 전형적인 컨트롤러 통합 테스트 템플릿은 다음과 같다.

    ```java
    @SpringBootTest
    @AutoConfigureMockMvc
    @AutoConfigureRestDocs  // Spring REST Docs를 사용할 때만 필요
    @Import(RestDocsConfig.class)  // Spring REST Docs를 사용할 때만 필요
    @ActiveProfiles("test")
    public class MemberControllerTest {

        @Autowired
        private MockMvc mvc;

        @Autowired
        private ObjectMapper objectMapper;

        private JacksonTester<MemberDto> jsonMemberDto;


        @BeforeEach
        public void beforeEach() {
            JacksonTester.initFields(this, objectMapper);
        }


        @DisplayName("정상적인 회원 테스트")
        @Test
        public void create_normal_member() throws Exception {
            MemberDto memberDto = new MemberDto( ... );
            MockHttpServletResponse response = mvc.perform(
                    post("/api/members")
                        .contentType(MediaType.APPLICATION_JSON)
                        .accept(MediaTypes.HAL_JSON)
                        .content(objectMapper.writeValueAsString(event)))
                    .andDo(print())
                    .andDo(document("create-member"))  // Spring REST Docs 적용 시에만 필요
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.id").exists())
                    .andExpect(jsonPath("$.email").value("homo.efficio@gmail.com"));
        }
    ```
