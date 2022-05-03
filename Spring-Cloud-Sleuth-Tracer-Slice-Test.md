# Spring Cloud Sleuth Tracer Slice Test

Spring Cloud Sleuth 는 쉽게 말해 요청 처리 흐름을 추적할 수 있는 traceId 와 spanId 를 쉽게 생성/관리하 수 있게 해주고, 로그에도 출력되게 하는 라이브러리다.

얻는 이익에 비해 설정은 굉장히 간편하므로 안 쓸 이유가 별로 없..기는 한데, 실행 시간을 줄이기 위한 슬라이스 테스트 할 때는 Tracer 를 주입해줘야 하는 데 이게 검색해도 잘 안 나온다.

어떻게 해야할까?

아래와 같이 Tracer 를 생성해서 Bean 으로 생성해주고,

```kotlin
package aaa.bbb.ccc

import brave.Tracing
import org.springframework.boot.test.context.TestConfiguration
import org.springframework.cloud.sleuth.Tracer
import org.springframework.cloud.sleuth.brave.bridge.BraveBaggageManager
import org.springframework.cloud.sleuth.brave.bridge.BraveTracer
import org.springframework.context.annotation.Bean

@TestConfiguration
class TracerConfig {

    @Bean
    fun tracer(): Tracer {
        val tracer = Tracing.newBuilder().build().tracer()
        return BraveTracer(
            tracer,
            BraveBaggageManager(),
        )
    }
}
```

테스트 코드에서는 아래와 같이 사용할 수 있다.

```kotlin
@WebMvcTest(XXXController::class)
@Import(
    // 테스트에 필요한 클래스들
    TracerConfig::class,
)
internal class XXXControllerTest {
```

참고로 traceId 를 구할 때 `currentSpan()`이 null 일 수도 있으므로 아래와 같은 처리를 해주는 것이 좋다

```
val traceId = (tracer.currentSpan() ?: tracer.nextSpan()).context().traceId()
```

