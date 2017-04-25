# Spring - [ ]가 포함된 URL 파라미터 바인딩 하기

스프링에서 Servlet Request에 포함된 parameter들의 모델 객체(또는 DTO 객체)로의 바인딩은 `ServletRequestDataBinder`에서 담당한다.

큰 흐름을 살펴보면 다음과 같다.

1. parameterName을 key로, parameterValue를 value로 해서 모든 parameter를 `MutablePropertyValues`에 넣은 후, 
2. `MutablePropertyValues`에 저장된 값을 `DataBinder`를 통해 모델 객체(또는 DTO 객체)로 바인딩힌다.

## 문제

parameterName이 특별한 점 없이 그냥 일반적이라면 모든 과정이 행복하게 끝나는데, parameterName이 아래와 같이 

>items[0][count]

같은 형식으로 들어오면 다음과 같은 에러를 만나게 되는데, 더 안타까운 것은 이 에러는 `BindingResult`로는 잡히지도 않는다는 점이다.

```
org.springframework.beans.InvalidPropertyException: 
  Invalid property 'items[0][count]' of bean class [어쩌구DTO]: 
    Property referenced in indexed property path 'items[0][count]' is neither an array nor a List nor a Map

`items[0][count]`가 가리키는 값이 배열도, 리스트도, 맵도 아니라서 예외 발생
```

그런데 저런 형식의 데이터가 들어올까?

들어온다. jQuery에서 다음과 같이 데이터를 서버에 보내면,

```javascript
$.ajax({
    url: '어쩌구-서버-API',
    contentType: 'application/json',
    method: 'GET',
    crossDomain: true,
    data: {
        id: "321",
        items: [
            {
                id: "abc987",
                count: 3
            }
        ],
        emails: ['abc@abc.com', 'sdf@sdf.com']
    }
}).done(function() {
    // 성공 시 처리 
});
```

다음과 같은 URL로 서버에 요청이 전달된다.

```
GET /어쩌구-서버-API?id=321&items%5B0%5D%5Bid%5D=abc987&items%5B0%5D%5Bcount%5D=3&emails%5B%5D=abc%40abc.com&emails%5B%5D=sdf%40sdf.com
```

URL Decoding하면 다음과 같다.

```
GET /어쩌구-서버-API?id=321&items[0][id]=abc987&items[0][count]=3&emails[]=abc@abc.com&emails[]=sdf@sdf.com
```

이처럼 실제로는 들어올 수 있는 데이터 형식인데, 이런 형식은 스프링이 자연스럽게 모델 객체로 바인딩 해주지 못한다.

## 해결

그래서 이를 보완할 수 있는 유틸 메서드를 만들어봤다.

```java
public static <T> T getDTOFromParamMap(Map<String, String[]> parameterMap, Class<T> dto) 
        throws IllegalAccessException, InstantiationException {

    final MutablePropertyValues sourceProps = getPropsFrom(parameterMap);

    T targetDTO = dto.newInstance();
    DataBinder binder = new DataBinder(targetDTO);
    binder.bind(sourceProps);

    return targetDTO;
}

private static MutablePropertyValues getPropsFrom(Map<String, String[]> parameterMap) {
    
    final MutablePropertyValues mpvs = new MutablePropertyValues();

    parameterMap.forEach(
            (k, v) -> {
                String dotKey =
                        k.replaceAll("\\[]", "")
                         .replaceAll("\\[(\\D+)", ".$1")
                         .replaceAll("]\\[(\\D)", ".$1")
                         .replaceAll("(\\.\\w+)]", "$1");
                mpvs.addPropertyValue(dotKey, v);
            }
    );

    return mpvs;
}
```

핵심 로직은 중간의 람다식 안의 정규표현식에 담겨 있는데, 테스트 코드를 보면 금방 이해할 수 있다.

```java
@Test
public void object인덱스형_key를_dot형으로_변환() throws Exception {

    String k = "items[0][options][1][a][12][b][abc][c][1234][c1][33][_1][___][99][a33][b3][aa3]";

    String result =
            k.replaceAll("\\[]", "")
             .replaceAll("\\[(\\D+)", ".$1")
             .replaceAll("]\\[(\\D)", ".$1")
             .replaceAll("(\\.\\w+)]", "$1");

    assertThat(result, is("items[0].options[1].a[12].b.abc.c[1234].c1[33]._1.___[99].a33.b3.aa3"));
}
```

쉽게 말해 `[]`로 구성된 parameterName을 스프링이 이해할 수 있는 `.` 형식으로 적절하게 변환해서 `MutablePropertyValues`에 에러 없이 집어넣을 수 있게 하고, `DataBinder`를 통해 모델 객체로 바인딩 하게 해준다.

단, 한 가지 제약 조건이 있는데 **[] 안에 들어가는 parameterName이 숫자로 시작하면 안된다**는 점이다.


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
