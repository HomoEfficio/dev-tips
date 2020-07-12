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
      AtomicInteger index = new AtomicInteger();
      return s.chars()
              .filter(c -> index.incrementAndGet() % 2 == 0)  // 여기!!
              .mapToObj(Character::toString)
              .collect(Collectors.joining(""));
  }
```

