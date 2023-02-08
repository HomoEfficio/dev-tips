# Cache 패턴

저장되어서는 안 될 값이 캐시에 저장되면 수동으로 캐시를 삭제해줘야 한다.
이를 위해 캐시 클라이언트로 직접 붙어 삭제하거나 운영용 api를 만들어 두고 유사 시 api를 통해 해당 캐시를 삭제하는 방식을 사용한다.

## Self Destruction Pattern

자멸 패턴이라고 이름 붙여 본다 ㅋ

예를 들어 어떤 캐시 키에 null이 들어있는 경우,  
캐시 유효기간 동안 또는 운영용 api를 통해서든 다른 이벤트에 의해서든 제거되기 전까지 계속 캐시에서 읽어서 null이 반환되게 하는 것보다는,  
캐시에서 null이 반환 됐을 때 캐시를 삭제해서 이후 조회 시 캐시가 아닌 DB에서 조회해서 다시 캐시에 저장되게 하는 것이 실무적으로 편리하다.

구현 슈도코드는 다음과 같다.

```kotlin
    fun getAnyInfo(key: String): AnyInfo {
        // 캐시에서 먼저 읽고 없으면 DB를 조회해서 캐시에 넣고 반환하는 @Cacheable 메서드
        val anyInfo = anyService.getInfo(key)

        // 캐시에 null 이 들어가 있거나 유효하지 않은 값이 들어있는 경우 캐시 삭제해서 자동 치유
        if (anyInfo == null || anyInfo.isValid == false) {
            cacheManager.getCache("any-cache-name")!!
                .evictIfPresent(key)
        }

        return anyInfo
    }
```

