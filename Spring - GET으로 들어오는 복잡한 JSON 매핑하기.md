# Spring - GET으로 들어오는 복잡한 JSON 매핑하기

중첩된 객체와 배열을 포함하고 있는 JSON을 POST로 받으면 `@RequestBody`를 붙인 DTO 또는 모델 객체에 쉽게 매핑할 수 있다.

그런데 GET으로 받으면 쉽게 매핑이 안 된다.

하지만 Spring의 `DataBinder`와 `MutablePropertyValues`를 활용하면 GET으로 받은 복잡한 JSON도 POST로 받은 것 만큼이나 쉽게 매핑할 수 있다.

```java
public static <T> T getDTOFromParameterMap(Map<String, String[]> parameterMap, Class<T> dto) throws IllegalAccessException, InstantiationException {

        final MutablePropertyValues source = new MutablePropertyValues();

        parameterMap.forEach(
                (k, v) -> {
                    String dotKey = k.replaceAll("\\[(\\D+)", ".$1")
                            .replaceAll("(\\.\\w+)]", "$1");
                    source.addPropertyValue(dotKey, v);
                }
        );

        T targetDTO = dto.newInstance();
        DataBinder binder = new DataBinder(targetDTO);
        binder.bind(source);

        return targetDTO;
    }
```
