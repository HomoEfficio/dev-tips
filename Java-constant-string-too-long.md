# Java constant string too long

화면 단에서 `console.log(JSON.stringify(jsonObject))`로 JSON 문자열을 따와서 API 테스트에 사용할 때가 있다.

그런데 JSON 문자열의 크기가 크면(정확한 값은 나중에 찾아보기로), 컴파일 시 다음과 같이 짤막한 에러 메시지가 출력된다.

```
Error:(165, 16) java: constant string too long
```

결국 어떤 한도를 넘는 긴 문자열은 그냥 따옴표 안에 넣어서 문자열 상수로 만들어서 사용할 수 없다.

이럴 때는 **문자열 상수 대신에 파일에 해당 JSON 문자열을 저장하고, 파일에서 읽으면 문제 없이 사용할 수 있다.**

스프링이라면 `resources` 폴더에 있는 파일을 다음과 같이 읽어서 문자열로 반환할 수 있다.

```java
Resource sharedDataSourcesJson = resourceLoader.getResource("classpath:SharedDataSourcesJson");
Stream<String> lines = Files.lines(Paths.get(sharedDataSourcesJson.getURI()));
return lines.collect(Collectors.joining("\n"));
```

https://www.baeldung.com/spring-classpath-file-access 여기에 다양한 방법이 나와있으며,  
https://www.baeldung.com/reading-file-in-java 이것도 참고할만하다.

또는 Jackson으로 다음과 같이 간단하게 Deserialize 할 수 있다.

```java
ObjectMapper mapper = new ObjectMapper();
Abc abc = mapper.readValue(string, Abc.class);
```




