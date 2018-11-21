# Spring Data JPA 테스트 시 auto-increment 문제

Spring Data JPA 를 사용하는 프로젝트에서 서비스 클래스를 테스트 할 때 auto-increment 로 인한 문제가 발생할 수 있다.

## 문제 상황

다음과 같은 테스트 메서드들이 있을 때, 첫번째 실행되는 메서드에서 생성되는 레코드의 id 는 1L 이므로 테스트는 문제 없이 통과한다.


```java
    @After
    public void teardown() {
        this.portfolioRepository.deleteAll();
    }

    @Test
    @Transactional
    public void 포트폴리오에_속한_첨부파일_조회() throws IOException {
        // given
        createPortfolio();

        // when
        Portfolio portfolio = this.portfolioService.findById(1L);
        List<Attachment> attachments = portfolio.getAttachments();

        // then
        assertThat(portfolio.getTitle()).isEqualTo("포트폴리오 제목 1");
        assertThat(attachments.size()).isEqualTo(2);
        assertThat(attachments.get(0).getOrgFileName()).isEqualTo("원래 파일 이름 1");
        assertThat(attachments.get(0).getSeq()).isEqualTo(0);
        assertThat(attachments.get(1).getFileType()).isEqualTo(Attachment.FileType.IMAGE);
    }

    @Test
    @Transactional
    public void 포트폴리오_좋아요() throws IOException {
        // given
        createPortfolio();
        Portfolio portfolio = this.portfolioService.findById(1L);
        User user = UserFactory.ofNormal();
        this.userRepository.save(user);

        // when
        LikeResponse likeResponse = this.portfolioService.likePortfolio(portfolio.getId(), user.getEmail());
        String json = likeResponseJacksonTester.write(likeResponse).getJson();
        System.out.println(json);


        // then
        assertThat(json).isEqualTo("{\"portfolioId\":1,\"likeBy\":[\"nickname1\"]}");
    }

    @Test
    @Transactional
    public void 포트폴리오_좋아요_취소() throws IOException {
        // given
        createPortfolio();
        Portfolio portfolio = this.portfolioService.findById(1L);
        User user1 = UserFactory.ofNormal();
        this.userRepository.save(user1);
        this.portfolioService.likePortfolio(portfolio.getId(), user1.getEmail());
        User user2 = UserFactory.ofDefault();
        this.userRepository.save(user2);
        this.portfolioService.likePortfolio(portfolio.getId(), user2.getEmail());

        // when
        LikeResponse likeResponse = this.portfolioService.unlikePortfolio(1L, user1.getEmail());
        String json = likeResponseJacksonTester.write(likeResponse).getJson();

        // then
        assertThat(json).isEqualTo("{\"portfolioId\":1,\"likeBy\":[\"nickname2\"]}");
    }
```

하지만 두 번째 실행되는 메서드는 실패한다. teardown()에서 `this.portfolioRepository.deleteAll();`를 통해 **이전 테스트에서 생성된 레코드를 지우더라도, auto-increment 에 의해 두 번째 실행되는 메서드에서 생성되는 레코드의 id는 2L 이 된다.**

따라서 `Portfolio portfolio = this.portfolioService.findById(1L);` 이 부분에서 원하는 레코드를 찾을 수 없어 테스트가 실패하게 된다. 실패한 메서드 한 개만 단독으로 실행해보면 테스트가 통과하는데 이는 단 하나의 메서드만 실행하고 끝나므로 auto-increment의 영향을 받을 일이 없기 때문이다. 실제로는 이렇게 개별 메서드 별로 하나하나 테스트 할 수는 없으므로 통과해도 의미 없다.

## 해결 방법

이렇게 auto-increment에 의한 문제가 테스트의 발목을 잡으면 **테스트에 사용하는 데이터베이스에 맞게 auto-increment 값을 재지정**해주면 된다. 

흔히 사용하는 H2 에서는 다음과 같이 해주면 된다.

```java
    @After
    public void teardown() {
        this.portfolioRepository.deleteAll();
        this.entityManagr
            .createNativeQuery("ALTER TABLE portfolio ALTER COLUMN `pofo_post_no` RESTART WITH 1")
            .executeUpdate();
    }
```

## 제약 사항

하지만 이 방법은 `@DataJpaTest` 등 `@Transactional`이 적용되는 서비스 클래스를 테스트할 때는 적용 가능하지만 `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`를 사용하는 통합 테스트에서는 사용할 수 없다.

`@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`를 사용하는 통합 테스트에서 위와 같이 `ALTER ~ RESTART WITH 1`을 실행하면 다음과 같은 에러가 발생한다.

>javax.persistence.TransactionRequiredException: Executing an update/delete query

Transaction을 적용하기 위해 다음과 같이 `@Transactional`을 붙여주면,

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Transactional  // resetAutoIncrement() 실행을 위해 필요
public class UserIntegrationTest {
```

`teardown()`에서 `deleteAll()`을 해도 제대로 사라지지 않는 희한한 현상을 목격하게 된다. 

`@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`를 사용하면 테스트가 수행되는 main 스레드와 TestRestTemplate에 의해 발생된 request를 처리하는 스레드가 다르다는 것이 원인인 것으로 추정되는데, 여튼 `teardown()`에서 `deleteAll()`로 지워도 다음 테스트 메서드 수행시에는 특별히 insert 문이 실행되지 않았음에도 버젓이 레코드 하나가 남아있고, 다시 `teardown()`에서 `deleteAll()`로 지워도 다음 테스트 메서드 수행시에는 특별히 insert 문이 실행되지 않았음에도 버젓이 레코드 하나가 남아있고, .. 반복 .. 

이건 말로는 전달이 어렵고 디버그 모드 켜놓고 Watch에도 등록해놓고 실시간으로 봐야 어떤 제 맛이 날 것이다.

하지만 `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`과 `TestRestTemplate`을 사용하는 통합 테스트에서는 `findById(1L)` 같은 방식으로 테스트 코드를 짤 필요가 없으므로 auto-increment reset에 집착하지 않는다면 통합 테스트 작성 및 통과에 실질적인 문제는 없다.

## 다른 방법

### `@DirtyContext` 사용

테스트 클래스에 

```java
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
```
를 붙이면 된다는 글도 있는데, 이렇게 하면 auto-increment 문제는 발생하지 않지만, **테스트 메서드 하나하나 실행 완료될 때마다 애플리케이션 컨텍스트를 새로 만들기 때문에 테스트 실행 속도가 매우 느려지므로** auto-increment 문제 해결을 위해 `@DirtiesContext`를 사용하는 것은 권장할 만한 방법은 아니다.
