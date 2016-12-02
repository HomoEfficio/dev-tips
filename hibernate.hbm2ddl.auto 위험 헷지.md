# hibernate.hbm2ddl.auto 위험 헷지

JPA를 사용하면서 가장 편한 기능 중의 하나가 `hibernate.hbm2ddl.auto` 설정을 통해 Java 애플리케이션 단에서 엔티티 객체의 변경 사항이 DB 단에서의 테이블 스키마에 자동으로 반영된다는 점이다.

그런데 이는 '개발'의 편리성에 해당할 뿐이고, '운영' 중에는 `hibernate.hbm2ddl.auto` 설정 때문에 SW 시스템이나 서비스 분야에서 겪을 수 있는 가장 큰 규모의 장애를 마주하게 될 수도 있다. 바로 데이터가 몽창 사라지는 것이다.. 이보다 더 위험한 설정이 있을 수 있을까.. ㄷㄷㄷ

## 문제 상황

결론부터 말하면 여러가지 상황이 짬뽕되어 `hibernate.hbm2ddl.auto`의 값이 의도하지 않게 `create`로 적용되어서 Drop database 발생.. ㄷㄷㄷ

이제 그 '여러가지 상황'이 뭐였는지 차례차례 알아보자. 먼저 `hibernate.hbm2ddl.auto`의 옵션과 의미를 알아보자.

## hibernate.hbm2ddl.auto

http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#configurations-hbmddl

> `hibernate.hbm2ddl.auto`의 목적
> 
> - `SessionFactory`의 라이프 싸이클의 일부로서 `SchemaManagementTool` 액션을 자동으로 실행하게 한다.
> - 선택 가능한 옵션 목록은 `Action` enum의 `externalHbm2ddlName` 값으로 지정되어 있다.

선택 가능한 옵션 목록은 다음과 같다.

- `none` : 기본 값이며 아무 일도 일어나지 않는다.
- `create-only` : 데이터베이스를 새로 생성한다.
- `drop` : 데이터베이스를 drop 한다.
- `create` : 데이터베이스를 drop 한 후, 데이터베이스를 새로 생성한다.(기능적으로는 drop + create-only와 같다)
- `create-drop` : `SessionFactory`가 시작될 때 스키마를 drop하고 재생성하며, `SessionFactory`가 종료될 때도 스키마를 drop 한다.
- `validate` : 데이터베이스 스키마를 검증 한다.
- `update` : 데이터베이스 스키마를 갱신 한다.

## Spring 설정 정보 프로파일과 기본값

스프링은 아래와 같이 여러가지 환경 설정 정보를 프로파일을 통해 구분해서 적용할 수 있다.

```yml
# 이거슨 default
spring.jpa:
  database: MYSQL
  showSql: false
  properties.hibernate.dialect: org.hibernate.dialect.MySQL5InnoDBDialect
  properties.hibernate.hbm2ddl.auto: create

...

---
# 이거슨 local
spring.profiles: local
spring.jpa:
  database: MYSQL
  showSql: false
  properties.hibernate.dialect: org.hibernate.dialect.MySQL5InnoDBDialect
  properties.hibernate.hbm2ddl.auto: update

...

---
# 이거슨 상용
spring.profiles: production
spring.jpa:
  database: MYSQL
  showSql: false
  properties.hibernate.dialect: org.hibernate.dialect.MySQL5InnoDBDialect
  properties.hibernate.hbm2ddl.auto: update
  
...

```

위에서는 그냥 `...`라고 축약했지만 실제로는 많은 양의 환경 설정 정보가 있어서 default 프로파일에서 다른 프로파일까지의 물리적인 거리는 멀다. 그리고 보통은 프로파일을 명시하기 때문에 default의 설정은 신경 안 쓰기도 한다.

하지만 default 프로파일에는 설정되어 있는데 특정 프로파일에는 설정되어 있지 않은 설정 정보는 default의 값이 사용 된다. 

여기까지 알고 나니 위에 있는 `application.yml` 예에서 뭔가 심각하게 음산한 분위기가 느껴진다.

## 문제의 시작 0 - `properties.hibernate.hbm2ddl.auto: update`의 삭제

개발 환경에서는 `hibernate.hbm2ddl.auto`의 값을 `update`로 둘 수도 있겠지만, 상용 운영 환경에서는 권장되지 않는다. 이유는 http://egloos.zum.com/gyumee/v/2483659 에 잘 나와있다.

그런데 위의 `application.yml` 예에 보면 production 프로파일에 `properties.hibernate.hbm2ddl.auto: update`라고 되어 있다. 상용에 `update`라니.. 이럼 안 되자나.. 하면서 **production 프로파일에 있던 `properties.hibernate.hbm2ddl.auto: update`를 아예 지워버렸다**.

## 문제 1 - default 환경 설정 정보의 상속

production 프로파일에서 `properties.hibernate.hbm2ddl.auto: update`를 지울 때는 이제 `update`가 적용되지 않고 `none`이 적용될 것이다..라는 의도였겠지만, **실제로는 `properties.hibernate.hbm2ddl.auto`의 값은 default 프로파일에 지정된 값을 상속받아 사용**하게 된다. 그 값은 `create`다. 데이터베이스를 drop하고 다시 생성한다는 바로 그 `create`.. 그 다음은..

## 문제 2 - DB 권한

사실 아무리 애플리케이션 단에서 위와 같이 위험하게 설정이 되어 있더라도, 애플리케이션이 DB를 사용하는 계정에 데이터베이스를 drop 할 권한이 없으면 데이터베이스가 날라가는 일은 발생하지 않는다.

그런데, 그냥 개발 편의만을 생각하면서 진행하다가 운영 환경에서도 이런 권한 설정을 꼼꼼히 챙기지 않는다면..

## 예방책

`application.yml`을 잘 설정하면 되는 것 아니냐.. 맞다. 그 '잘'이라는게 언제나 '잘' 되지는 않으므로 예방책으로 충분하지는 않다. 좀 더 안전하게 문제 발생을 막을 방법이 필요하다.

## 스프링 환경 설정 정보 우선 순위 활용

스프링 환경 설정 정보 우선 순위가 `application.yml` 보다 높은 곳에서 `hibernate.hbm2ddl.auto`의 값을 올바르게 지정해주면 문제를 막을 수 있다.

스프링 환경 설정 정보 우선 순위는 http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html 에 나와있다.

테스트에 사용되는 것을 제외하면 App 실행 시 `-D` 옵션으로 값을 지정해주는 것이 우선 순위가 가장 높다. 따라서 다음과 같은 조치가 효과적일 수 있다.

>운영 환경에서는 애플리케이션 실행 명령 옵션으로
>`-Dspring.jpa.properties.hibernate.hbm2ddl.auto=none` 지정

## DB 권한 설정

애플리케이션 단 뿐 아니라 DB 단에서도 보완하는 것이 좋다.

>애플리케이션에서 DB에 접근하는 계정에는 필요한 권한만 부여하고, 위험한 명령에 대한 권한은 부여하지 않는다.

## 기타

이 외에도 실행 시에 `application.yml`을 파싱해서 읽어들이는 시점에 프로파일 별 `hibernate.hbm2ddl.auto`의 값을 검사하는 방법도 있겠다. 

예를 들어, local 프로파일에서는 `hibernate.hbm2ddl.auto`의 값을 검사하지 않지만, 운영 프로파일에는 `hibernate.hbm2ddl.auto`의 값이 `none`이 아니면 애플리케이션의 실행을 중단하고 ERROR를 표시하도록 뭔가를 만들어서 적용하면 문제 발생 원인도 알 수 있어서 좋을 것 같다.

이 외에도 여러가지가 있을 것이다.

## 정리

`hibernate.hbm2ddl.auto`는 개발 편의성을 대폭 높여주는 기능을 제공하지만, 그만큼 강력하고 위험하기도 하다. 조심하고 또 조심하되, 아래와 같은 예방책도 적용하자.

>1. 운영 환경에서는 애플리케이션 실행 명령 옵션으로
>`-Dspring.jpa.properties.hibernate.hbm2ddl.auto=none` 지정
>
>2. 애플리케이션에서 DB에 접근하는 계정에는 필요한 권한만 부여

## 더 읽을 거리

- http://egloos.zum.com/gyumee/v/2483659
- http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#configurations-hbmddl
- http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
