# Java Stream 에서 index 사용하는 법

Stream API 에서 index 를 사용할 수 있게 해주는 언어도 있지만 자바는 그렇지 않은 것 같다.

그래도 우회 방법은 있다.

다음 예제는 `"123456789"` 에서 짝수번째에 위치한 글자만을 모아서 출력하는 예제다.

```java
  @Test
  void indexWhileStreaming() {
      String nums = "123456789";
      assertThat(filtered(nums)).isEqualTo("2468");
  }

  private String filtered(String s) {
      int[] index = {0};
      return s.chars()
              .filter(c -> ++index[0] % 2 == 0)  // 여기!!
              .mapToObj(Character::toString)
              .collect(Collectors.joining(""));
  }
```

---
올리고 나니 여러분께서 좋은 의견을 주셨는데, 관련 예제가 https://www.baeldung.com/java-collections-zip 여기에 아주 잘 나와있으니 같이 보면 좋겠다.

참고로 D3.js 에서는 대략 아래와 같은 방식으로 아주 편리하게 index 를 사용할 수 있는 API 를 제공해준다.

```java
  private String filtered(String s) {
      return s.chars()
              .filter((c, i) -> i % 2 == 1)  // (data, index) 를 파라미터로 하는 람다식을 쓰면 data에는 데이터가 index에는 인덱스 값이 알아서 들어온다.
              .mapToObj(Character::toString)
              .collect(Collectors.joining(""));
  }
```

Kevin Lee 님께서 알려주신 아래 방법도 괜찮아 보인다.

```java
  private String filtered(String s) {
      return IntStream.iterate(0, i -> i + 1)  // IntStream.iterate() 와 limit() 을 사용해서 for 문 같아 보이는 함수형 식 사용
          .limit(s.length())
          .filter(i -> (i & 1) == 1)
          .mapToObj(i -> Character.toString(s.charAt(i)))
          .collect(Collectors.joining());      
  }
```
