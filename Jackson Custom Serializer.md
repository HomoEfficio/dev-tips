# 알고보면 만만한 Jackson Custom Serialization

API 서버를 만들다보면 어떤 객체를 JSON으로 만들때, 특정 필드만 제외하거나 특정 필드의 이름을 바꿔야 하는 일이 생길 수 있다.

Java 기반 API 서버라면 JSON 처리를 위해 [Jackson](https://github.com/FasterXML/jackson)을 많이 사용하는데, Jackson으로 Java 객체를 JSON으로 Serialize 할 때 앞에서 말한 것 처럼 커스터마이징 하는 방법은 검색해보면 꽤나 다양하게 많이 나오는데, 

- 뭔 필터를 만들고, 프로바이더를 꽂고 어쩌고 자시고 지지고 볶고 태우는 방법도 있고, 
- Jackson을 커스터마이징 할 생각은 아예 포기하고, 그 대신 JSON 화 할 대상 객체 A를 커스터마이징해서 별도의 객체 B를 다시 만들고, B를 대상으로 그냥 `objectMapper.writeValueAsString(B)`로 시원하게 처리하는 방법도 있고 ㅋㅋ,
- 기타 등등 다양하다.

Jackson을 기준으로, 개인적으로 생각할 때 가장 직관적이어서 이해하기 쉽고, 유지보수 하기도 쉽고, 코드량도 적은 커스터마이징 방법을 적어본다.

## 얼개

Jackson을 이용한 JSON Serialization은 결국 `objectMapper.writeValueAsString(객체)`고, 따라서 `objectMapper`에 Custom Serializer를 어떤 식으로 집어 넣느냐 하는 문제인데, 이와 관련된 구조만을 추려서 요약하면 다음 그림과 같다.

![Imgur](http://i.imgur.com/JUYnjRE.png)

`objectMapper`는 여러 개의 `module`을 가질 수 있고, `module`도 여러 개의 `customSerializer`를 가질 수 있다.

## 절차

이 얼개를 바탕으로 serialization을 커스터마이징하려면 다음의 절차가 필요하다.

1. 실제 커스터마이징을 담당하는 customSerializer를 만든다
2. customSerializer를 module에 추가한다.
3. module을 objectMapper에 등록한다.

결국 `customSerializer`만 **잘** 만들어주면 된다. 참 쉽다 ㅋㅋ.

### CustomSerializer 만들기

JSON serialization 과정에서 커스터마이징이라고 할 수 있는 것은, 결국 앞에서 말한대로 특정 필드의 의도적인 제외, 특정 필드 이름의 변경 정도다. 그래서 만들어야 할 코드는 대략 아래 정도일 것 같다.

```java
필드1이름지정(field1Key);
필드1값지정(field1Value);

필드2이름지정(field2Key);  // 필드2를 serialization에서 제외하려면 이 행을 아예 안 써버리면 된다.
필드2값지정(field2Value);  // 필드2를 serialization에서 제외하려면 이 행을 아예 안 써버리면 된다.

필드3이름지정(field3KeyWhateverYouWant);  //필드3의 이름을 바꾸려면 원하는 이름으로 지정해주면 된다.
필드3값지정(field3Value);  // 필드3의 값은 원래 그대로 써준다.

...
```

CustomSerializer를 만들려면 `JsonSerializer` 클래스를 상속해야 한다. 그리고 IDE를 통해 Override 해야할 메서드를 자동 생성하면 대략 아래와 같은 코드가 자동 생성 된다.

```java
package homo.efficio.json.selective.serialization.jackson;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonSerializer;
import com.fasterxml.jackson.databind.SerializerProvider;
import homo.efficio.json.selective.serialization.domain.FamilyMember;

import java.io.IOException;

/**
 * Created by HomoEfficio on 2016-10-22.
 */
public class FamilyMemberSerializer extends JsonSerializer<FamilyMember> {

    @Override
    public void serialize(FamilyMember value, JsonGenerator gen, SerializerProvider serializers) throws IOException, JsonProcessingException {
        
    }
}
```

이제 앞에서 예상했던 코드를 JsonSerializer가 정해준 방식에 맞게 구현해주면 되는데, 그 방식이라는게 아주 간단하다.

``` java
@Override
public void serialize(FamilyMember value, JsonGenerator gen, SerializerProvider serializers) throws IOException, JsonProcessingException {

    gen.writeStartObject();

    gen.writeFieldName("id");
    gen.writeString(String.valueOf(value.getId()));

    gen.writeFieldName("name");
    gen.writeString(value.getName());

    gen.writeFieldName("cellPhone");
    gen.writeObject(value.getCellPhone());

    gen.writeFieldName("children");
    gen.writeObject(value.getChildren());

    gen.writeEndObject();
}
```

글로 바꿔써보면 그냥 JSON 문자열을 만드는 것과 똑같다.

>- `gen.writeStartObject()`로 `{`를 열고,
>  - `gen.writeFieldName(fieldName)`로 필드 이름을 쓰고,
>    - serialize 할 값이 primitive라면 `gen.writeString(primitive)`로 값을 쓰고,
>     - serialize 할 값이 reference라면 `gen.writeObject(reference)`로 값을 쓰고,
>- `gen.writeEndObject()`로 `}`를 닫는다.
>
>`""`, `:`, `,`, `[]` 등은 고맙게도 Jackson이 알아서 처리해준다능..

당연한거 아닌가.. 근데 필터나 프로바이더 방식은 이렇게 당연해보이지 않고 복잡하더라능..

이제 방금 만든 `customSerializer`를 `module`에 추가해보자.

### module 만들고 customSerializer 추가

`customSerializer`는 만들었는데, `module`은 어떻게 만들어야 하지?

간단하다. Jackson은 `SimpleModule`이라는 클래스를 제공해주며, 아래의 코드가 전부다.

```java
SimpleModule simpleModule = new SimpleModule();
```

`module`에 `customSerializer`를 추가하는 코드도 간단하다. 하지만 주석에 표시한 의미는 이해하고 넘어가자.

```java
// FamilyMember 클래스는 FamilyMemberSerializer로 Serialize 하겠다는 의지의 표현.
simpleModule.addSerializer(FamilyMember.class, new FamilyMemberSerializer());
```

### objectMapper 만들고 module 추가

`objectMapper`는 익숙하니까 설명할 것도 없다. 그냥 코드를 보자.

```java
ObjectMapper objectMapper = new ObjectMapper();

SimpleModule simpleModule = new SimpleModule();
// FamilyMember 클래스는 FamilyMemberSerializer로 Serialize 하겠다는 의지의 표현.
simpleModule.addSerializer(FamilyMember.class, new FamilyMemberSerializer());

objectMapper.registerModule(simpleModule);
```

### 실제 사용

```java
objectMapper.writeValueAsString(serialize_할_객체);
```

## 실전 배치

더는 설명할 것도 없다. 잔소리 따위는 접고 그냥 코드를 보면 바로 이해된다. ㅋㅋ

https://gist.github.com/HomoEfficio/e3cee0071f0ce84ed6d7791d0410d8d5

## 정리

>Jackson으로 JSON Serialize 할 때, 필드 이름 변경, 특정 필드 제외 등 커스터마이징이 필요하면,
>- `JsonSerializer`를 상속받아 `customSerializer`를 만들고,
>  - `gen.writeStartObject()`, `gen.writeFieldName()`, `gen.writeString()/gen.writeObject()`, `gen.writeEndObject()`로 serialize 로직을 원하는 대로 구성해서,
>- `SimpleModule`에 `customSerializer`를 추가하고,
>- `objectMapper`에 `simpleModule`를 등록하고,
>- `objectMapper.writeValueAsString(serialize_할_객체)`로 serialize 하면 된다.


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
