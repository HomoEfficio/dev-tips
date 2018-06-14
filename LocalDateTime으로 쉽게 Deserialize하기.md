# LocalDateTime으로 쉽게 Deserialize하기

## 문제

```json
{
  "periodStartDateTime": "2018-05-01 00:00:00"
}
```

위와 같이 그냥 `yyyy-MM-dd HH:mm:ss`와 같은 문자열을 Java8의 `LocalDateTime` 타입으로 Deserialize하려면 어떻게 해야할까?

## 시도 및 실패

```java
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime periodStartDateTime;
```

이렇게 `@JsonFormat`으로 포맷만 지정해주면 될 것 같지만, 실행하면 다음과 같은 에러가 난다.

```
...
Caused by: com.fasterxml.jackson.databind.JsonMappingException: Can not instantiate value of type [simple type, class java.time.LocalDateTime] from String value ('2018-05-01 00:00:00'); no single-String constructor/factory method
...
```

`LocalDateTime`에는 일정 형태의 문자열을 받아서 `LocalDateTime` 객체를 생성하는 생성자나 팩토리 메서드가 없다고 투덜댄다.

## 해결

이럴 땐 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310' 의존 관계를 build.gradle 이나 pom.xml 에 추가해주면 된다.

결국 정리하면

1. 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310' 의존 관계를 build.gradle 이나 pom.xml 에 추가한다.

    ```groovy
    compile('com.fasterxml.jackson.datatype:jackson-datatype-jsr310')
    ```

1. 아래와 같이 `@JsonFormat` 애노테이션으로 문자열 규격을 정해준다.

    ```java
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime periodStartDateTime;
    ```

끝.
