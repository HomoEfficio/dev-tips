# Spring Data JPA 테스트 시 auto-increment 문제

Spring Data JPA 를 사용하는 프로젝트에서 서비스 클래스를 테스트 할 때 auto-increment 로 인한 문제가 발생할 수 있다.

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

하지만 두 번째 실행되는 메서드는 실패한다. teardown()에서 `this.portfolioRepository.deleteAll();`를 통해 이전 테스트에서 생성된 레코드를 지우더라도, auto-increment 에 의해 두 번째 실행되는 메서드에서 생성되는 레코드의 id는 2L 이 된다.

따라서 `Portfolio portfolio = this.portfolioService.findById(1L);` 이 부분에서 원하는 레코드를 찾을 수 없어 테스트가 실패하게 된다. 하지만 실패한 메서드 한 개만 단독으로 실행하면 통과한다.

이럴 때는 테스트에 사용하는 데이터베이스에 맞게 auto-increment 값을 재지정해주면 된다. 흔히 사용하는 H2 에서는 다음과 같이 해주면 된다.

```java
    @After
    public void teardown() {
        this.portfolioRepository.deleteAll();
        this.entityManagr
            .createNativeQuery("ALTER TABLE portfolio ALTER COLUMN `pofo_post_no` RESTART WITH 1")
            .executeUpdate();
    }
```

검색해보면 테스트 클래스에 

```java
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
```
를 붙이면 된다고 하는데, 이렇게 하면 auto-increment 문제는 발생하지 않지만, 테스트 메서드 하나하나 실행 완료될 때마다 애플리케이션 컨텍스트를 새로 만들기 때문에 테스트 실행 속도가 매우 느려지므로 auto-increment 문제 해결을 위해 `@DirtiesContext`를 사용하는 것은 권장할만한 방법은 아니다.
