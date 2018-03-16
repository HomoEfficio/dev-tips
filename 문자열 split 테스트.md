# 문자열 split 테스트

A.split(B).length 결과는 A에 B만 달랑 있을 때만 0 이다.

```java
	@Test
	public void split_테스트() throws Exception {
		final String splitter = ":::";

		assertThat(splitter.split(splitter).length).isEqualTo(0);

		assertThat("a:::".split(splitter).length).isEqualTo(1);
		assertThat("a:::".split(splitter)[0]).isEqualTo("a");

		assertThat(":::a".split(splitter).length).isEqualTo(2);
		assertThat(":::a".split(splitter)[0]).isEqualTo("");
		assertThat(":::a".split(splitter)[1]).isEqualTo("a");

		assertThat("abc".split(splitter).length).isEqualTo(1);
		assertThat("abc".split(splitter)[0]).isEqualTo("abc");

		assertThat("".split(splitter).length).isEqualTo(1);
		assertThat("".split(splitter)[0]).isEqualTo("");
	}
```
