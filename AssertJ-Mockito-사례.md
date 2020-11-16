# AssertJ-Mockito 사례

테스트 코드 작성에 사용했던 AssertJ 및 Mockito 사례를 모아본다.

## verify

### 특정 필드값을 가진 객체를 인자로 전달하는 메서드를 verify 할 때

```java
Mockito.verify(XXXRepository).save(ArgumentMatchers.argThat((XXX xxx) -> xxx.getYYY().equals(zzz)));
```

## assertThat

### 어떤 리스트가 특정 범위 안의 값만 포함하는지 assert 할 때

```java
Assertions.assertThat(randomNumbers)
    .containsOnlyElementsOf(otherNumbers);
```


