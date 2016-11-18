# 조금은 신경써줘야 하는 Jackson Custom Deserialization

[알고보면 만만한 Jackson Custom Serialization](https://github.com/HomoEfficio/dev-tips/blob/master/%EC%95%8C%EA%B3%A0%EB%B3%B4%EB%A9%B4%20%EB%A7%8C%EB%A7%8C%ED%95%9C%20Jackson%20Custom%20Serializer.md)에 이어 이번에는 Jackson Custom Deserialzation을 알아보자.

Serialzation과 Deserialization은 대칭 관계니까 언뜻 생각하기엔 별로 다르지 않을 것 같은데, 당연하지만 세부적인 과정에서는 대칭이 아니기 때문에, Deserialization에서는 대수롭지 않긴 하지만 성능 상 고려 해야할 점이 한 가지 있다.

## Deserialization 결과로 생성되는 객체

먼저 Deserialize 후 생성되는 객체부터 살펴보자.

```java
public class FamilyMember {

    private Long id;
    private String name;
    private CellPhone cellPhone;
    private Set<FamilyMember> children;

    ... // 이하 생성자, getter/setter, equals() 등 생략
}

public class CellPhone {

    private String number;
    private MobileVendor vendor;

    ... // 이하 생성자, getter/setter, equals() 등 생략
```

`FamilyMember`는 `id`, `name` 같은 단순 필드 외에 `CellPhone`이라는 타입의 객체와, 자기와 같은 타입의 Set을 포함하고 있다.

## Serializtion과 Deserialization의 차이

>`FamilyMember`가 `CellPhone`을 포함하는 관계가 이미 반영되어 있는 객체를 JSON 문자열로 변환하는 Serialization과는 달리, 
>
>JSON 문자열을 `FamilyMember`가 `CellPhone`을 포함하는 관계가 존재하는 객체로 변환하는 과정에서는, 
>- **포함된 객체를 Deserialize 할 때마다 `ObjectMapper`가 하나 씩 더 필요하다.** 

성능 상 고려해야할 점이라는 것이 바로 추가적으로 필요한 `ObjectMapper`다. `ObjectMapper`를 생성하는 과정에서 조금만 신경을 써주면 된다.

### 신경 쓰지 않고 Serialization와 같은 방식으로 작성한 코드

[`FamilyMemberSerializer` 클래스](https://gist.github.com/HomoEfficio/e3cee0071f0ce84ed6d7791d0410d8d5#file-1-familymemberserializer-java)에서는 아무런 필드 없이 모든 것을 `serialze()` 메서드 안에서 처리했다. 마찬가지 방식으로 `FamilyMemberDeserializer`를 작성하면 아래와 같다. 

```java
public class FamilyMemberDeserializer extends JsonDeserializer<FamilyMember> {

    @Override
    public FamilyMember deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {

        ObjectCodec objectCodec = p.getCodec();
        JsonNode jsonNode = objectCodec.readTree(p);

        ObjectMapper objectMapper = new ObjectMapper();
        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addDeserializer(CellPhone.class, new CellPhoneDeserializer());
        objectMapper.registerModule(simpleModule);

        final Long id = jsonNode.get("id").asLong();
        final String name = jsonNode.get("name").asText();
        final CellPhone cellPhone = objectMapper.convertValue(jsonNode.get("cellPhone"), CellPhone.class);
        final Set<FamilyMember> children = objectMapper.convertValue(jsonNode.get("children"), new TypeReference<LinkedHashSet<FamilyMember>>() {
        });

        return new FamilyMember(id, name, cellPhone, children);
    }
}
```

하지만, 위 코드에는 약간의 실수가 묻어있다.

실수를 바로잡은 코드는 다음과 같다.

### 신경 쓴 코드

```java
public class FamilyMemberDeserializer extends JsonDeserializer<FamilyMember> {

    private final ObjectMapper objectMapper;

    public FamilyMemberDeserializer() {
        this.objectMapper = new ObjectMapper();
        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addDeserializer(CellPhone.class, new CellPhoneDeserializer());
        this.objectMapper.registerModule(simpleModule);
    }

    @Override
    public FamilyMember deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {

        ObjectCodec objectCodec = p.getCodec();
        JsonNode jsonNode = objectCodec.readTree(p);

        final Long id = jsonNode.get("id").asLong();
        final String name = jsonNode.get("name").asText();
        final CellPhone cellPhone = objectMapper.convertValue(jsonNode.get("cellPhone"), CellPhone.class);
        final Set<FamilyMember> children = objectMapper.convertValue(jsonNode.get("children"), new TypeReference<LinkedHashSet<FamilyMember>>() {
        });

        return new FamilyMember(id, name, cellPhone, children);
    }
}
```

### 차이 비교

차이가 보이는가? 

`ObjectMapper`를 `deserialize()` 안에서 반복해서 생성하면 성능에 악영향을 미치므로 `FamilyMember` 생성 시 한 번만 생성해서 재사용하는 것이 좋다. `customDeserializer`에 필요한 `SimpleModule`과 `CustomDeserializer`도 마찬가지다.

## 정리

>- 단순한 primitive 값 필드 뿐 아니라 다른 객체를 필드로 포함하고 있는 객체의 Deserialzation 과정에서는,
>- 그 다른 객체도 Deserialze 해야 하기 떄문에 `ObjectMapper`가 또 필요하다.
>- 이 `ObjectMapper`를 `deserialize()` 메서드 실행 시 마다 생성하지 않도록 주의하자.

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
