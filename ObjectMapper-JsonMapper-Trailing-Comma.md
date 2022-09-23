# Trailing Comma 허용하는 ObjectMapper, JsonMapper

### ObjectMapper

```kotlin
private val objectMapper = ObjectMapper(
    JsonFactory.builder()
        .enable(JsonReadFeature.ALLOW_TRAILING_COMMA)
        .build()
)
```

### JsonMapper

```kotlin
private val jsonMapper = JsonMapper.builder(
    JsonFactory.builder()
        .enable(JsonReadFeature.ALLOW_TRAILING_COMMA)
        .build()
).build()
```
