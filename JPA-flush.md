# JPA 강제 flush 가 필요한 순간

Hibernate 5.4.32 버전 사용 중인데 신기한 현상이 발견됐다.

심사 승인(approve)이 이뤄지면 심사 승인 행위가 히스토리에 저장되고,  
승인에 의해 발행(publish)이 이뤄지는 시나리오다.

```kotlin
val dbXxx = xxxRepository.findXXXById(xxxId)
dbXxx.approve()

val dbYyy = yyyRepository.findYYYById(yyyId)
dbYyy.publish()

return XxxOut.fromXxxEntity(dbXxx)  // DTO 반환, 히스토리 1개가 히스토리 목록에 2번 포함됨
```

승인을 하면 심사 승인 히스토리는 DB에 1건만 생성이 된다. 그런데 그 동일한 1건이 위 DTO 반환 시점에서는 희한하게도 히스토리 목록에 2번 포함된다.

재미있는 현상은 코드에 전혀 손대지 않고 `approve()` 바로 다음 행에 BreakPoint 만 걸어서 잠시 실행을 멈췄다가 재개하면 히스토리 목록에 정상적으로 1번만 포함된다.

정확한 원인은 알 수 없지만 `approve()` 직후 강제로 flush()를 해주면 이런 현상은 사라진다.

```kotlin
val dbXxx = xxxRepository.findXXXById(xxxId)
dbXxx.approve()
xxxRepository.flush()  // 강제 flush

val dbYyy = yyyRepository.findYYYById(yyyId)
dbYyy.publish()

return XxxOut.fromXxxEntity(dbXxx)  // DTO 반환, 히스토리 1개는 히스토리 목록에도 1개만 포함됨
```
