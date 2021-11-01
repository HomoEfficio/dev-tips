# @Transactional(noRollbackFor)

스프링에서 선언적으로 편리하게 Tx 관리를 할 수 있게 해주는 마법 같은 @Transactional 애노테이션이 있다.

기본 propagation 값이 PROPAGATION_REQUIRED 라서 @Transactional 이 붙어 있는 메서드가 호출될 때 이미 시작된 Tx가 있다면 새 Tx를 만들지 않고 이미 시작된 Tx에 참여한다.

여기까지는 좋은데 중간에 예외가 발생한다면 어떻게 될까?

```kotlin
@Service
class OuterService(
    // InnerService를 주입받는다
) {
    @Transactional
    fun outerMethod() {
        try {
            // db insert 1
            // db update 2
            innerService.innerMethod()
        } catch (Exception e) {
            println("예외지만 되던지지 않는다")
        }
    }
    ...    
}


@Service
class InnerService {

    @Transactional
    fun innerMethod() {
        // db insert 3
        // db update 4
        if (1 == 1) throw RuntimeException()  여기!!
    }    
}
```

위와 같이 기존 Tx에 참여하는(participating) innerMethod() 에서 'db insert 3', 'db update 4' 후에 RuntimeException 이 발생하고,  
outerMethod()에서 예외를 잡아서 되던지지 않으면 어떻게 될까?

1. 결국 RuntimeException이 outerMethod() 밖으로 던져지지 않으므로 1, 2, 3, 4 모두 롤백되지 않는다.  
2. RuntimeException가 발생한 innerMethod()에서 수행된 3, 4는 롤백되고, outerMethod()에서 수행된 1, 2는 롤백되지 않는다.  
3. 어쨌든 예외가 발생했으므로 1, 2, 3, 4 모두 롤백된다.

정답은 3번이다.

이유는 짧게 얘기하면 기본 설정이 그렇게 동작하도록 돼있기 때문이다.  
자세하고 정확한 이유는 https://github.com/spring-projects/spring-framework/blob/main/spring-tx/src/main/java/org/springframework/transaction/support/AbstractPlatformTransactionManager.java#L233 여기 주석에 나와 있고, 검색하면 많이 나오니까 간단히 줄이고 실제 사용할 때 어떻게 해야하는지에 집중해서 조금 더 알아보자.


# Case 1 - 1,2,3,4 모두 롤백 되지 않게

Tx에 참여하는 메서드에 `@Transaction(noRollbackFor = RuntimeException.class)`를 붙여주면 아무것도 롤백되지 않는다.

innerMethod()에서는 RuntimeException을 던지더라도 `@Transaction(noRollbackFor = RuntimeException.class)`로 지정된 롤백 정책에 의해 롤백이 되지 않고, 롤백 마킹도 하지 않는다.

롤백 마킹도 되지 않았으므로 예외를 잡아서 되던지지 않는 outerMethod()에서도 롤백이 발생하지 않는다.

```kotlin
@Service
class OuterService(
    // InnerService를 주입받는다
) {
    @Transactional
    fun outerMethod() {
        try {
            // db insert 1
            // db update 2
            innerService.innerMethod()
        } catch (Exception e) {
            println("예외지만 되던지지 않는다")
        }
    }
    ...    
}


@Service
class InnerService {

    @Transactional(noRollbackFor = RuntimeException.class)  // 여기!!
    fun innerMethod() {
        // db insert 3
        // db update 4
        if (1 == 1) throw RuntimeException()
    }    
}
```


# Case 2 - 1,2 는 롤백되고, 3,4는 롤백 되지 않게

Outer인 1,2는 롤백되고, Inner인 3,4가 롤백 되지 않게 하려면,  
Outer는 예외를 잡아서 RuntimeException을 되던져야 하고,  
Inner를 별도의 Tx에서 실행되게 해야한다.

```kotlin
@Service
class OuterService(
    // InnerService를 주입받는다
) {
    @Transactional
    fun outerMethod() {
        try {
            // db insert 1
            // db update 2
            innerService.innerMethod()
        } catch (Exception e) {
            // println("예외지만 되던지지 않는다")
            throw RuntimeException("되던진다")  // 여기!!
        }
    }
    ...    
}


@Service
class InnerService {

    @Transactional(
        noRollbackFor = RuntimeException.class,
        propagation = Propagation.REQUIRES_NEW,  // 여기!!
    )
    fun innerMethod() {
        // db insert 3
        // db update 4
        if (1 == 1) throw RuntimeException()
    }    
}
```



# Case 3 - 1,2,3,4 모두 롤백 되게

innerMethod(), outerMethod() 모두에 디폴트 @Transactional 만 붙이면 된다.

innerMethod()에서 발생한 RuntimeException이 3,4의 롤백을 유발하고 기본 정책에 따라 롤백 마킹이 되고,  
롤백 마킹이 됐기 때문에 outerMethod()는 RuntimeException을 되던지든 말든 관계 없이 롤백 된다.

```kotlin
@Service
class OuterService(
    // InnerService를 주입받는다
) {
    @Transactional
    fun outerMethod() {
        try {
            // db insert 1
            // db update 2
            innerService.innerMethod()
        } catch (Exception e) {
            println("예외지만 되던지지 않는다")
        }
    }
    ...    
}


@Service
class InnerService {

    @Transactional
    fun innerMethod() {
        // db insert 3
        // db update 4
        if (1 == 1) throw RuntimeException()
    }    
}
```

