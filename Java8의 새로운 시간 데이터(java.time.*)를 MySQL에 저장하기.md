# Java8의 새로운 시간 데이터(java.time.\*)를 MySQL에 저장하기 

Java8에는 시간 데이터를 더 편리하게 처리할 수 있게 해주는 `LocalDate`, `LocalDateTime` 등의 클래스들이 `java.time` 패키지에 추가되었다.

하지만, 위 클래스들로 표현되는 시간 데이터를 별다른 처리 없이 JPA를 이용해서 MySQL에 저장하면 버전에 따라서 아래와 같은 에러가 날 수도 있다.

```java
...
Caused by: com.mysql.jdbc.MsqlDataTruncation: Data truncation: Incorrect dateme value: '\xAC\xED\x00\x05sr\x0Djava.time.Ser\x95]\x84\xBA\x1B"H\xB2\x0C\0\x00xpw\x07\x03\x00\x00\x07\xE0\x05\x1Fx' for column 'start_date' at row 1
...
```
또는 MySQL에 원하던 바와 다르게 `tinyblob` 타입으로 저장되기도 한다.

어느 경우든, MySQL의 데이터 타입과 뭔가 궁합이 안 맞는 모양이다. 중간에 Converter 같은 것이 필요할 것 같다.

## Attribute Converter

JPA 2.1 부터 `Attribute Converter`라는 기능이 도입되었다. `AttributeConverter` 클래스를 상속받는 자체 Converter를 만들면, Java8의 시간 데이터 타입을 JPA에서 인식할 수 있는 타입으로 자동으로 변환되게 할 수 있다.
아래는 `java.time.LocalDate`에 대한 Converter다. 

```java
import javax.persistence.AttributeConverter;
import javax.persistence.Converter;
import java.time.Instant;
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.Date;

@Converter(autoApply = true)
public class LocalDatePersistenceConverter implements AttributeConverter<LocalDate, Date> {

    @Override
    public Date convertToDatabaseColumn(LocalDate entityValue) {

        return Date.from(entityValue.atStartOfDay().atZone(ZoneId.systemDefault()).toInstant());
    }

    @Override
    public LocalDate convertToEntityAttribute(Date databaseValue) {
    
        return Instant.ofEpochMilli(databaseValue.getTime()).atZone(ZoneId.systemDefault()).toLocalDate();
    }
}

```

자체 Converter를 만든다고 끝은 아니다. 어느 데이터에 이 Converter를 적용할지 지정해줘야 한다. 아래와 같이 Converter에 의한 자동변환이 필요한 데이터에 `@Convert` 애노테이션을 지정해준다.

```java
@Convert(converter = LocalDatePersistenceConverter.class)
@Column(name = "start_date")
private LocalDate startDate;
```


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
