# Spring Data JPA 에서 Java8 Date/Time(JSR-310) 사용하기 

Java8에는 시간 데이터를 더 편리하게 처리할 수 있게 해주는 `LocalDate`, `LocalDateTime` 등의 클래스들이 `java.time` 패키지에 추가되었다. 날짜/시간 차이 계산, 비교, 년/월/일/시/분/초 단위 별 추출 등 풍부한 기능을 제공해주므로 사용성이 아주 좋다. [Joda-Time](http://www.joda.org/joda-time/)을 사용하고 있었다면, Java8에서는 `java.time (JSR-310)`으로 마이그레이션 하는 것이 좋다.

하지만, `java.time (JSR-310)`를 별다른 처리 없이 JPA를 이용해서 MySQL에 저장하면 버전에 따라서 아래와 같은 에러가 날 수도 있다.

```java
...
Caused by: com.mysql.jdbc.MsqlDataTruncation: Data truncation: Incorrect dateme value: '\xAC\xED\x00\x05sr\x0Djava.time.Ser\x95]\x84\xBA\x1B"H\xB2\x0C\0\x00xpw\x07\x03\x00\x00\x07\xE0\x05\x1Fx' for column 'start_date' at row 1
...
```
또는 날짜/시간을 나타내는 컬럼이 MySQL에서 `datetime` 타입이 아니라 `tinyblob` 타입으로 생성되어서 아래와 같이 원치 않는 형식으로 저장되기도 한다.

![](http://i.imgur.com/oXXnfnO.png)

어느 경우든, JPA와 Java8 Date/Time은 뭔가 조치를 취해주지 않으면 원하는 대로 쓸 수 없다.

데이터의 생성/수정 시각을 기록하는 JPA Auditing을 대상으로 그 조치 방법을 알아보자. 모든 엔티티가 상속해야 하는 `BaseEntity`라는 추상 클래스를 만들어서 이 클래스에 JPA Auditing을 적용하는 상황이다.

## Jsr310JpaConverters.class를 활용하는 방법 - 1

**Spring Data JPA 1.8 이상부터 사용가능한 방법**으로, 아마 가장 간단한 방법일 것 같다.

아래와 같이 **`@EntityScan`에 `Jsr310JpaConverters.class`를 지정**해주기만 하면 된다. 다만, `Jsr310JpaConverters.class`를 사용하지 않았다면 굳이 지정해 주지 않아도 자동 설정으로 처리될 **`basePackages`도 명시적으로 지정해줘야만 엔티티를 로딩할 수 있다는 단점**이 있다.

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.convert.threeten.Jsr310JpaConverters;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@EnableJpaAuditing
@EntityScan(
        basePackageClasses = {Jsr310JpaConverters.class},  // basePackageClasses에 지정
        basePackages = {"homo.efficio.toy.member.domain"}) // basePackages도 추가로 반드시 지정해줘야 한다
@SpringBootApplication
public class MemberApplication {

    public static void main(String[] args) {
        SpringApplication.run(MemberApplication.class, args);
    }
}
```

엔티티에도 날짜/시간형 필드에 `@Temporal(TemporalType.TIMESTAMP)`를 붙여주지 않아도 된다. 사실은 붙이고 싶어도 붙일 수가 없다. [Java EE API 문서](http://docs.oracle.com/javaee/7/api/javax/persistence/Temporal.html)에 보면 `@Temporal`은 `java.util.Date`이나 `java.util.Calendar`에만 붙일 수 있게 되어 있다.

```java
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import javax.persistence.*;
import java.io.Serializable;
import java.time.LocalDateTime;

@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class BaseEntity implements Serializable {

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdDateTime;

    @LastModifiedDate
    @Column(name = "last_modified_at", updatable = true)
    private LocalDateTime lastModifiedDateTime;
}
```

위의 코드를 보면 `BaseEntity`에는 따로 무슨 조치를 취한 것 없이 깔끔하다.

`Jsr310JpaConverters` 클래스는 사실 다음에 설명할 Attribute Converter를 활용하는 방법을 Spring에서 구현해서 쓰기 편하게 Wrapping 해준 `Jsr310Converters` 클래스를 JPA에서 사용할 수 있게 해주는 클래스다.


## Jsr310JpaConverters.class를 활용하는 방법 - 2

역시 **Spring Data JPA 1.8 이상부터 사용가능한 방법**으로 전체 애플리케이션에 영향을 주지 않고 개별 필드 단위로 적용해야할 때 사용할 수 있다.

`@EntityScan(basePackageClasses= ...)`를 지정할 필요 없이 변환하고자 하는 필드에만 `@Convert(converter = Jsr310JpaConverters.LocalDateTimeConverter.class)`를 붙여주면 된다. `LocalDateTime`뿐아니라 `LocalDate`, `LocalTime`, `LocalDateTime`, `Instant`, `ZoneId` 모두에 대한 변환 기능이 구현되어 있으므로, 필드 타입에 따라 알맞은 구현체를 `converter`로 지정해주면 된다.

```java
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class BaseEntity implements Serializable {

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    @Convert(converter = Jsr310JpaConverters.LocalDateTimeConverter.class)
    private LocalDateTime createdDateTime;

    @LastModifiedDate
    @Column(name = "last_modified_at", updatable = true)
    @Convert(converter = Jsr310JpaConverters.LocalDateTimeConverter.class)
    private LocalDateTime lastModifiedDateTime;
}
```

개별 필드 단위로 적용가능하지만, 이는 바꿔말하면 변환해야하는 개별 필드에는 모두 붙여줘야 한다는 얘기이기도 하다. 따라서 특별한 이유가 있는 것이 아니라면 Jsr310JpaConverters.class를 활용하는 방법 - 1 을 활용해서 `LocalDateTime` 등 Java8의 Date/Time을 사용하는 모든 필드에 일괄 적용되게 하는 것이 좋다.



## Attribute Converter를 활용하는 방법

JPA 2.1 부터 `Attribute Converter`라는 기능이 도입되었다. **`AttributeConverter` 클래스를 상속받는 자체 Converter를 만들면, Java8의 날짜/시간 데이터 타입을 JPA에서 인식할 수 있는 타입으로 자동으로 변환**되게 할 수 있다.
아래는 `java.time.LocalDateTime`에 대한 Converter다.

```java
import javax.persistence.AttributeConverter;
import javax.persistence.Converter;
import java.time.Instant;
import java.time.LocalDateTime;
import java.util.Date;

import static java.time.Instant.ofEpochMilli;
import static java.time.LocalDateTime.ofInstant;
import static java.time.ZoneId.systemDefault;

@Converter(autoApply = true)
public class LocalDateTimePersistenceConverter implements AttributeConverter<LocalDateTime, Date> {

    @Override
    public Date convertToDatabaseColumn(LocalDateTime localDateTime) {
        return Date.from(localDateTime.atZone(systemDefault()).toInstant());
    }

    @Override
    public LocalDateTime convertToEntityAttribute(Date date) {
        return ofInstant(ofEpochMilli(date.getTime()), systemDefault());
    }
}

```

위의 코드는 `LocalDateTime`에 대한 구현체만 들어있는데, 앞에서 언급한 `org.springframework.data.convert.Jsr310Converters` 클래스는 `LocalDate`, `LocalTime`, `LocalDateTime`, `Instant`, `ZoneId` 모두에 대한 변환 기능을 구현해서 제공해주며, Spring Data JPA 1.8 이상이라면 앞에서 살펴본 것처럼 `Jsr310JpaConverters`를 통해 Spring Data JPA에서 사용할 수 있다.

자체 Converter를 만들었다고 끝난 것이 아니다. 어느 데이터에 이 Converter를 적용할지 지정해줘야 한다. 따라서 아래와 같이 `BaseEntity`에서 **Converter에 의한 자동변환이 필요한 데이터에 `@Convert` 애노테이션을 지정**해준다.

```java
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import javax.persistence.*;
import java.io.Serializable;
import java.time.LocalDateTime;

@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class BaseEntity implements Serializable {

    @CreatedDate
    @Convert(converter = LocalDateTimePersistenceConverter.class)  // <- @Converter를 지정 해줘야 한다.
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdDateTime;

    @LastModifiedDate
    @Convert(converter = LocalDateTimePersistenceConverter.class)  // <- @Converter를 지정 해줘야 한다.
    @Column(name = "last_modified_at", updatable = true)
    private LocalDateTime lastModifiedDateTime;
}
```

## getter, setter를 변형해서 활용하는 방법

이 방법은 `Jsr310Converters`이나 `AttributeConverter` 등에 포함된 변환 로직을 그냥 getter, setter에 직접 심어버리는 방법으로 가장 직관적이고 간단하며, 외부 의존성도 적다. **Spring을 사용하지 않는다면 이 방법으로 해결하면 된다.** 다만, 타이핑 양은 좀 되지만 복붙신공이면 될 일이고.. ㅋㅋ

```java
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import javax.persistence.*;
import java.io.Serializable;
import java.time.LocalDateTime;
import java.util.Date;

import static java.time.Instant.ofEpochMilli;
import static java.time.LocalDateTime.ofInstant;
import static java.time.ZoneId.systemDefault;
import static java.util.Objects.isNull;

@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class BaseEntity implements Serializable {

    @CreatedDate
    @Temporal(TemporalType.TIMESTAMP)  // java.util.Date이므로 @Temporal을 붙여준다.
    @Column(name = "created_at", updatable = false)
    private Date createdDateTime;

    @LastModifiedDate
    @Temporal(TemporalType.TIMESTAMP)  // java.util.Date이므로 @Temporal을 붙여준다.
    @Column(name = "last_modified_at", updatable = true)
    private Date lastModifiedDateTime;



    // 변환 로직을 직접 구현

    public LocalDateTime getCreatedDateTime() {
        return getLocalDateTimeFrom(createdDateTime);
    }

    public void setCreatedDateTime(final LocalDateTime createdDateTime) {
        this.createdDateTime = getDateFrom(createdDateTime);
    }

    public LocalDateTime getLastModifiedDateTime() {
        return getLocalDateTimeFrom(lastModifiedDateTime);
    }

    public void setLastModifiedDateTime(final LocalDateTime lastModifiedDateTime) {
        this.lastModifiedDateTime = getDateFrom(lastModifiedDateTime);
    }

    private LocalDateTime getLocalDateTimeFrom(Date date) {
        return isNull(date) ? null : ofInstant(ofEpochMilli(date.getTime()), systemDefault());
    }

    private Date getDateFrom(LocalDateTime localDateTime) {
        return isNull(localDateTime) ? null : Date.from(localDateTime.atZone(systemDefault()).toInstant());
    }
}
```

## 정리

Java8의 `java.time (JSR-310)` 데이터를 DB에 저장해야 한다면,

>- Spring Data JPA 1.8 이상이라면 `Jsr310JpaConverters` 클래스를 사용하자.
>
>- Spring Data JPA 1.8 미만이지만, JPA 2.1 이상이라면 `Jsr310Converters` 클래스를 사용해서 `AttributeConverter`를 상속하고 변환 로직을 구현해서 사용하자.
>
>- 이도 저도 아니라면 `java.time (JSR-310)`과 `java.util.Date`를 상호 변환할 수 있는 로직을 `getter/setter` 안에 직접 구현해서 사용하자.

참고로 `@EnableJpaAuditing`은 소스 루트에 있는 루트 애플리케이션 클래스(`@SpringBootApplication`이 붙어있는 클래스)에만 붙여주면 테스트에도 JPA Auditing이 모두 적용된다.

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
