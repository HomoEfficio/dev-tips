# Spring Boot 3 + Colima 에서 MySQL TestContainer 사용

Docker 환경을 구성하기 위해 [Colima](https://github.com/abiosoft/colima)를 사용한다고 가정

간단히 요약하면 기본적으로 다음 4가지 설정이 필요하다.
>- DOCKER_HOST
>- TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE
>- TESTCONTAINERS_HOST_OVERRIDE
>- TESTCONTAINERS_REUSE_ENABLE

## TestContainers 설정

### 가장 간단한 방법

`~/.zshrc` 파일에 다음과 같이 환경 변수를 등록하면 된다.

```shell
export DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock"
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock
export TESTCONTAINERS_HOST_OVERRIDE=$(colima ls -j | jq -r '.address')
export TESTCONTAINERS_REUSE_ENABLE=true
```

### 설정 위치를 다르게 하려면

설정값은 [TestContainers 문서](https://java.testcontainers.org/features/configuration/#configuration-locations)에 다음 우선 순위로 설정 위치를 지정할 수 있다고 한다.

  >1. Environment variables
  >2. `.testcontainers.properties` in user's home folder. Example locations:
  >  - Linux: /home/myuser/.testcontainers.properties
  >  - Windows: C:/Users/myuser/.testcontainers.properties
  >  - macOS: /Users/myuser/.testcontainers.properties
  >3. testcontainers.properties on the classpath.

추가로 gradle 등 빌드 파일에서 임시 환경 변수를 등록할 수도 있다.

`DOCKER_HOST`의 경우 다음과 같이 경로를 지정해야 하는데,

```
DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock"
```

HOME 디렉터리를 `~/.zshrc` 파일에서는 위와 같이 `${HOME}`으로 간단히 구할 수 있지만,  
`~/.testcontainers.properties` 파일이나 `classpath:testcontainers.properties` 파일에서는 하드코딩 밖에는 방법이 없고,  
`build.gradle`에서는 구할 수는 있지만 번거로운 스크립트를 작성해야 한다.

따라서 **다른 것은 몰라도 `DOCKER_HOST`는 `~/.zshrc`에서 환경 변수로 등록하는 것이 현실적으로 가장 편리하다.**  
다른 값도 애플리케이션마다 다르게 지정할 이유가 사실 없으므로 그냥 모두 `~/.zshrc`에서 환경 변수로 등록하는 것이 좋다.

## MySQL 설정

### src/test/resources/application.yml

```yml
spring:
  application.name: YOUR_APPLICTION_NAME
  sql.init:
    platform: mysql  # src/test/resources/ 에 있는 schema-mysql.sql, data-mysql.sql 파일을 자동으로 읽어서 실행 
    mode: always
```

## Test class

- 여러 테스트에서 하나의 스프링 테스트 컨텍스트를 재사용하기 위해 만든 AbstractIntegrationTest 에 MySQL 테스트컨테이너 설정을 추가한다.

```java
@Testcontainers  // 여기!!
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class AbstractIntegrationTest {

    // ...

    private static final String USERNAME = "username";
    private static final String PASSWORD = "password";
    private static final String DATABASE_NAME = "edp";

    @Container  // 수동으로 mySQLContainer.start(), mySQLContainer.stop() 할 필요 없게 해주는 애너테이션
    // MySQLContainer 가 AutoCloseable 인터페이스를 구현하고 있으므로
    // suppress 해주지 않으면 컴파일러 warning 발생
    // TestConainers 가 내부적으로 cleanup 을 해주므로 suppress만 해주면 된다
    @SuppressWarnings("resource")
    static final MySQLContainer<?> mySQLContainer = new MySQLContainer<>("mysql:8")
        .withReuse(true)  // 테스트 메서드마다 컨테이너 생성하지 않고 한 번 생성해서 재사용
        .withUsername(USERNAME)
        .withPassword(PASSWORD)
        .withDatabaseName(DATABASE_NAME);

    @DynamicPropertySource
    public static void overrideProps(DynamicPropertyRegistry dynamicPropertyRegistry) {
        dynamicPropertyRegistry.add("spring.datasource.url", mySQLContainer::getJdbcUrl);
        dynamicPropertyRegistry.add("spring.datasource.username", () -> USERNAME);
        dynamicPropertyRegistry.add("spring.datasource.password", () -> PASSWORD);
    }

    // ...
}
```

## 한 걸음 더

- 위와 같이 AbstractIntegrationTest의 필드로 선언하고 `withReuse(true)`로 지정하고 `environment("TESTCONTAINERS_REUSE_ENABLE", "true")`을 통해 환경 변수로 재사용을 지정해주면,
  - 테스트 메서드마다 MySQL 컨테이너가 1개씩 생성되는 것은 피할 수 있지만, 테스트 클래스마다 MySQL 컨테이너가 1개씩 생성되는 것은 피할 수 없다.
  - 테스트 로그에서 테스트 클래스 갯수만큼의 `Container mysql:8 started in OOOOO`가 반복 출력되며 이는 다수의 MySQL 컨테이너가 생성되는 것을 의미한다.
- MySQL 컨테이너를 하나 생성하고 연결이 가능해질 때까지 MacBook M1 Pro 32G Sonoma 14.6.1 기준으로 거의 10초 가까이 걸리므로 테스트 클래스마다 컨테이너를 생성/삭제하는 것은 생산성을 크게 떨어뜨린다.
- **여러 테스트 클래스에 걸쳐 하나의 MySQL 컨테이너만 사용하려면 어떻게 해야할까?**

### Singleton

- `하나만`하면 떠오르는 디자인 패턴은? 싱글턴(Singleton) 패턴을 써보자.

```java
public enum MySQLTestContainer {

    INSTANCE;

    @Container
    private final MySQLContainer<?> container;

    @SuppressWarnings("resource")
    MySQLTestContainer() {
        container = new MySQLContainer<>("mysql:8")
                .withReuse(true)
                .withUsername("username")
                .withPassword("password")
                .withDatabaseName("testdb");
    }

    public MySQLContainer<?> getContainer() {
        return container;
    }
}
```
- 위와 같이 싱글턴을 만들어 사용하면 테스트 실행 시 아래 에러가 발생한다.
- >java.lang.IllegalStateException: Mapped port can only be obtained after the container is started  
  >컨테이너가 아직 시작되지도 않았는데 컨테이너 사용을 시도하면서 발생하는 에러로 보인다.
- 따라서 MySQL 테스트 컨테이너를 사용하기 전에 명시적으로 `start()`를 해줘야하고 그러려면 `@Container`를 삭제해야 하고, 그러면 `stop()`도 명시적으로 해줘야 메모리 누수를 막을 수 있다.
- `start()`는 싱글턴 인스턴스 생성 시 호출하면 되므로 간단한데, `stop()`은 언제 어디에서 호출해야 할까?
- shutdownHook 에 추가해주는 것이 가장 좋을 것 같다.
- 개선하면 다음과 같다.

```java
public enum MySQLTestContainer {

    INSTANCE;

    // 여기! @Container 삭제
    private final MySQLContainer<?> container;

    @SuppressWarnings("resource")
    MySQLTestContainer() {
        container = new MySQLContainer<>("mysql:8")
                .withReuse(true)
                .withUsername("username")
                .withPassword("password")
                .withDatabaseName("testdb");
        container.start();  // 여기!!
    }

    public MySQLContainer<?> getContainer() {
        return container;
    }

    // 여기!!
    private void stopContainer() {
        if (container.isRunning()) {
            container.stop();
        }
    }

    static {
        Runtime.getRuntime().addShutdownHook(new Thread(INSTANCE::stopContainer));
    }
    // 여기!!
}


```

### AbstractIntegrationTest

- MySQLTestContainer 싱글턴을 사용하는 AbstractIntegrationTest는 아래와 같이 변경하면 된다.

```java
//    @Container
//    @SuppressWarnings("resource")
//    static final MySQLContainer<?> mySQLContainer = new MySQLContainer<>("mysql:8")
//        .withReuse(true)  // 테스트 메서드마다 컨테이너 생성하지 않고 한 번 생성해서 재사용
//        .withUsername(USERNAME)
//        .withPassword(PASSWORD)
//        .withDatabaseName(DATABASE_NAME);
    static final MySQLContainer<?> mySQLContainer = MySQLTestContainer.INSTANCE.getContainer();
```
- 테스트를 실행하고 로그를 확인해보면 전체 테스트에 대해 `Container mysql:8 started in OOOOO`가 딱 1번만 호출되는 것을 확인할 수 있다.
