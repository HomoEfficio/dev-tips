# Java8 LocalDateTime Jackson 직렬화 문제

Spring Boot 1.5.X 에서는 기본으로는 JSR-310을 지원하지 않는다.

따라서 `LocalDateTime`을 String으로 변환해줘야 하는데 가장 쉽고 널리 사용되는 방법은 `@JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")`를 붙여주는 방법이다. 실제 코드에서는 이렇게만 해주면 포맷으로 정한 String으로 ObjectMapper가 내부적으로 변환한 후에 직렬화를 해줘서 의도대로 잘 동작한다.

그런데 테스트 코드에서는 위와 같이 `@JsonFormat`을 붙여줘도, 정작 `objectMapper.writeValueAsString(dto)`를 실행하면 `LocalDateTime`이 정해진 포맷대로 String으로 변환되지 않고 다음과 같이 객체인 상태로 직렬화되고,

```java
LocalDateTime.of(2020, 01, 01, 0, 0, 0)
```

```json
{ 
  "lastPartDate":{ 
    "year":2020,
    "month":"JANUARY",
    "dayOfMonth":1,
    "dayOfWeek":"WEDNESDAY",
    "dayOfYear":1,
    "monthValue":1,
    "hour":0,
    "minute":0,
    "second":0,
    "nano":0,
    "chronology":{ 
      "id":"ISO",
      "calendarType":"iso8601"
    }
  }
}
```

다음과 같은 에러가 발생한다.

```java
"invalid request body(JSON parse error: Unexpected token (START_OBJECT), expected VALUE_STRING: Expected array or string.; nested exception is com.fasterxml.jackson.databind.JsonMappingException: Unexpected token (START_OBJECT), expected VALUE_STRING: Expected array or string.\n at [Source: java.io.ByteArrayInputStream@27f70b49; line: 1, column: 849] (through reference chain: a.b.c.d.XXXDto[\"lastPartDate\"]))"
```

이를 해결하는 방법도 다음과 같이 여러가지 있지만, 
- `jackson-datatype-jsr310`을 의존 관계로 추가해서 해결하는 방법: https://reflectoring.io/configuring-localdate-serialization-spring-boot/
- `AttributeConverter`를 이용하는 방법: https://homoefficio.github.io/2016/11/19/Spring-Data-JPA-에서-Java8-Date-Time-JSR-310-사용하기/

이것저것 설정할 것 없이 직접 간단하게 처리하려면 Custom Serializer를 구현하면 된다.

다음과 같이 Custom Serializer를 구현하고,

```java
public class CustomLocalDateTimeSerializer extends JsonSerializer<LocalDateTime> {

    private static DateTimeFormatter formatter =
            DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss");

    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider serializers) throws IOException, JsonProcessingException {
        gen.writeString(formatter.format(value));
    }
}
```

테스트 코드에서 ObjectMapper를 다음과 같이 설정해주면,

```java
  @Before
  public void setup() {
        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addSerializer(LocalDateTime.class, new CustomLocalDateTimeSerializer());
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(simpleModule);
        JacksonTester.initFields(this, objectMapper);
  }
```

다음과 같이 의도한대로 직렬화되고 에러도 발생하지 않는다.

```
{ "lastPartDate":"2020-01-01T00:00:00" }
```
