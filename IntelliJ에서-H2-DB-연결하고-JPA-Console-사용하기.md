# IntelliJ에서 H2 연결하고 JPA Console 사용하기

H2는 Web 콘솔도 지원해줘서 로컬 환경에서 웹을 통해 쉽게 데이터를 확인할 수 있다.   
IntelliJ **Ultimate 버전**에는 DB Client 도구가 포함돼있어서 로컬에서 H2 DB를 사용할 때 편리하게 사용할 수 있다.  
또한 JPA를 사용하는 경우 JPA Console을 사용할 수 있고, Hibernate Console처럼 JPA 구현체에 따른 콘솔도 지원한다.

스프링부트 애플리케이션에 H2를 임베디드 모드로 사용하는 케이스를 기준으로 한 번 시도해보자.

IntelliJ 2019.2.3, 자바 11, Gradle 5.6.2, 스프링부트 버전 2.2.0 RC1, H2 1.4 기준이고, 기본 `build.gradle`은 다음과 같다.

```groovy
...
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    runtimeOnly 'com.h2database:h2'
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    testImplementation 'org.springframework.security:spring-security-test'
}
...
```

## H2 웹 콘솔

H2는 기본적으로 웹 콘솔을 지원하며, 스프링부트에서는 다음의 2가지 방법으로 웹 콘솔을 활성화 할 수 있다. 

1. `spring-boot-devtools` 적용
1. application.properties 에 `spring.h2.console.enabled=true` 명시

기본 JDBC URL은 `jdbc:h2:mem:testdb` 이며, 스프링부트 애플리케이션을 실행하면 `localhost:8080/h2-console`을 통해 웹 콘솔에 접근할 수 있다.

![Imgur](https://i.imgur.com/V3jgpSb.png)

H2 웹 콘솔은 다음과 같이 여러 웹 브라우저로부터의 접근은 가능하지만,

![Imgur](https://i.imgur.com/7iSodqX.png)  
<https://www.h2database.com/html/tutorial.html#tutorial_starting_h2_console 그림 수정>

웹 콘솔이므로 당연한 말일 수도 있지만 다음과 같이 웹 브라우저가 아닌 다른 클라이언트로부터의 접근은 불가능하다.

![Imgur](https://i.imgur.com/ZY7UV9y.png)  
<https://www.h2database.com/html/tutorial.html#tutorial_starting_h2_console 그림 수정>

따라서 IntelliJ와 같은 다른 클라이언트로에서 접근하려면 먼저 H2 TCP 서버를 별도로 구동해야 한다.


## H2 TCP 서버 생성

### 의존 관계 설정 변경

H2 TCP 서버를 구동하려면 H2가 제공하는 라이브러리를 소스 코드 수준에서 사용해야하므로 H2를 더 이상 `runtimeOnly`로만 사용할 수 없다. 따라서 `build.gradle`에서 H2를 다음과 같이 `compile`로 변경해줘야 한다.

```groovy
...
dependencies {
    ...
    compile 'com.h2database:h2'
    ...
}
...
```

### H2 TCP 서버 구동 빈 추가

다음과 같이 H2 TCP 서버를 구동하는 빈을 추가한다. 스프링이 아니라면 빈 대신 별도의 Java 애플리케이션으로 작성해도 된다.

```java
package io.homo_efficio.learnmicroservicesspringboot.config;

import org.h2.tools.Server;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.sql.SQLException;

@Configuration
public class H2ServerConfig {

    @Bean
    public Server h2TcpServer() throws SQLException {
        return Server.createTcpServer().start();
    }
}
```

H2 TCP 서버의 기본 포트는 9092이며, 포트 등 기타 옵션은 `createTcpServer()`의 정의 부분을 보면 상세히 알 수 있다.

이제 스프링부트 애플리케이션을 실행하면 H2 TCP 서버가 함께 구동되고, 스프링부트 애플리케이션이 종료될 때 H2 TCP 서버도 함께 종료된다.

보통 H2는 개발용으로 로컬에서 사용하므로 빈 설정 시 다음과 같이 프로파일을 지정해두고 스프링부트 애플리케이션 구동 시 적절한 프로파일을 지정해서 실행하는 것이 좋다.

```java
@Profile("local")
@Configuration
public class H2ServerConfig {
...
```

이제 IntelliJ 쪽 설정을 알아보자.


## IntelliJ 데이터 소스 설정

다음과 같이 데이터베이스 설정 메뉴에서 H2를 선택하고,

![Imgur](https://i.imgur.com/UnochEY.png)

다음과 같이 H2 JDBC 드라이버를 지정한다.

![Imgur](https://i.imgur.com/1h22PXY.png)

다음과 같이 왼쪽 상단의 'Project Data Sources'에 표시된 항목을 클릭하고 URL, User를 입력한다. URL 입력 시 뭔가 오류를 표시하는 듯한 빨간색 세로 막대가 표시되는데 무시하면 된다.

![Imgur](https://i.imgur.com/H3ASiJO.png)

Test Connection을 클릭하면 다음과 같이 연결에 성공한다.

![Imgur](https://i.imgur.com/OjIh2EW.png)

H2 TCP 서버는 인메모리 모드, 파일 모드 둘 다 지원하며 URL 형식은 다음을 참고한다.

![Imgur](https://i.imgur.com/36vMBtK.png)  
<출처: https://h2database.com/html/features.html#database_url>

참고로 다음과 같이 IntelliJ의 드라이버 지정 화면에서도 Connection type 별로 다음과 같이 URL 템플릿을 친절하게 알려준다.

![Imgur](https://i.imgur.com/Rf6nGyG.png)

데이터 소스 설정 화면에서 Test Connection에 성공한 후 다음과 같이 'Schemas' 탭을 클릭해서 1초 정도 기다린 후 뜨는 목록에서 'PUBLIC' 스키마를 선택한다.

![Imgur](https://i.imgur.com/9F1V9ks.png)

OK를 클릭하면 IntelliJ 화면이 다음과 같이 바뀐다.

![Imgur](https://i.imgur.com/iM8KTCQ.png)

console 창에서는 다음과 같이 자동 완성 도움을 받으면서 쿼리를 작성하고 실행할 수 있다.

![Imgur](https://i.imgur.com/0tpkaJo.png)


## JPA Console 설정

IntelliJ Ultimate 버전에서는 DB Client 뿐만 아니라 JPA Console도 제공하며, 여기서 JPQL 쿼리를 직접 실행할 수 있다.

다음과 같이 'Project Structure' 화면에서 JPA Console을 사용할 모듈의 'main'에서 우클릭하고 JPA를 클릭한다.

![Imgur](https://i.imgur.com/Qo9F6Da.png)

다음과 같이 'Default JPA Provider'에서 실제 사용하는 JPA 구현체를 선택한다.

![Imgur](https://i.imgur.com/h6LUtjB.png)

다음과 같이 'Persistence' 탭을 클릭하고 'entityManagerFactory'를 펼치면 엔티티 클래스가 표시된다.

![Imgur](https://i.imgur.com/gVcR6zH.png)

다음과 같이 'entityManagerFactory'를 우클릭하고 'Assign Data Sources...'를 클릭한다.

![Imgur](https://i.imgur.com/7Rixe39.png)

다음과 같이 'Data Source'란을 클릭하면 나오는 목록에서 앞에서 설정한 데이터 소스를 선택하고 OK를 클릭한다.

![Imgur](https://i.imgur.com/PcIGAh2.png)

다음과 같이 'entityManagerFactory'에서 우클릭하고 'Console'을 클릭하면,

![Imgur](https://i.imgur.com/GPCA6zX.png)

다음과 같이 Console 선택 메뉴가 표시된다. JPA Console을 클릭하면,

![Imgur](https://i.imgur.com/wYRCu07.png)

다음과 같이 JPA Console이 화면에 표시된다.

![Imgur](https://i.imgur.com/6bpkFbW.png)

바로 JPQL을 입력해서 실행할 수 있다. 현재 스프링부트 애플리케이션이 실행 중이지 않아서 H2 TCP 서버가 기동 중인 상태가 아니므로 다음과 같은 에러가 발생한다.

![Imgur](https://i.imgur.com/OyeIBe8.png)

스프링부트 애플리케이션을 실행해서 H2 TCP 서버가 기동된 후에 다음과 같이 다시 JPQL을 실행하면 결과가 표시된다.

![Imgur](https://i.imgur.com/E5GlHhN.png)

### JPQL 이름 인식 문제

그런데 JPA Console에서 JPQL 실행 시 Java의 CamelCase 표기법을 snake_case 표기법으로 자동으로 변환하지 않아서 다음과 같이 엔티티 클래스 이름이나 필드 이름에 CamelCase 표기법이 사용된 경우 'not found' 에러가 난다.

다음 그림을 보면 CamelCase를 사용하면서도 `@Table(name = "MULTIPLICATION_ATTEMPT")`나 `@Column(name="MULTIPLICATION_ATTEMPT_ID")`를 명시해준 건 에러가 나지 않지만, `resultAttempt`처럼 CamelCase이면서도 `@Column`으로 이름을 지정해주지 않은 건 에러가 발생한다.

![Imgur](https://i.imgur.com/Bx022sX.png)

따라서 현재로는 JPA Console을 통해 JPQL을 문제 없이 사용하려면 CamelCase를 사용하는 엔티티 클래스 이름이나 필드 이름에는 `@Table`,`@Column`을 통해 실제 테이블에 사용될 snake_case 이름을 모두 지정해줘야 하는 불편함이 있다.


# 정리

>- IntelliJ Ultimate 버전에서는 DB Client를 사용할 수 있다.  
>- 스프링부트 애플리케이션에서는 H2 TCP 서버를 빈으로 띄우면 IntelliJ DB Client로 연결해서 사용할 수 있다.  
>- Project Structure에서 main 모듈에 JPA를 추가하고 데이터 소스를 설정해주면 JPA Console을 사용할 수 있다.  
>    - 엔티티 클래스 이름이나 필드 이름이 CamelCase로 작성된 경우 `@Table`, `@Column`으로 snake_case 이름을 모두 지정해줘야 한다.

