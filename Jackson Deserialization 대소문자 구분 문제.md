# Jackson Deserialization 대소문자 구분 문제

Jackson으로 json 문자열을 Java 객체로 Deserialization 할 때 property 이름 대소문자를 구별한다.

대략 아래와 같은 User 클래스가 있을 때,

```java
class User {
    private String name;
    private String email;

    // getter, setter
}
```

아래와 같이 JSON 문자열 내 property 이름이 `name`, `email`이 아니라 `Name`, `Email`로 되어 있으면, 

```java
objectMapper.readValue(
    "{ \"Name\":\"돌아이\", \"Email\":\"doleye@abc.com\"}",  // name, email이 아니라 Name, Email
    User.class
);
```

ObjectMapper는 기본적으로 대소문자를 구분하므로 아래과 같은 에러가 난다.

>com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException: Unrecognized field "Name" (class **.User), not marked as ignorable (2 known properties: "name", "email"])

해결은 간단하다. 아래와 같이 ObjectMapper에 `MapperFeature.ACCEPT_CASE_INSENSITIVE_PROPERTIES`를 `true`로 설정해주면 된다.

```java
new ObjectMapper().configure(MapperFeature.ACCEPT_CASE_INSENSITIVE_PROPERTIES, true);
```
