# CompletableFuture 사용 스타일

## Controller에서 `CompletableFuture.supplyAsync()` 사용

```java
@GetMapping("/something")
public CompletableFuture<ResponseEntity<ADto>> getSomething() {
    return CompletableFuture.supplyAsync(
        () -> ReponseEntity.ok(this.aService.getSomething()),
        this.customTaskExecutor
    );
}
 
@Service
public ADto getSomething() {
ADto a = getSomethingTakesLong();
return a;
}
```

기존 서비스 메서드가 결국 blocking 방식의 JDBC 드라이버로 DB에 질의하는 메서드라면 기존 서비스 메서드는 그대로 둔 채 컨트롤러에서만 비동기 처리를 하는 것도 나쁘진 않아 보인다.

하지만 서비스 메서드가 non-blocking으로 동작할 수 있다면 컨트롤러에서만 `CompletableFuture`를 쓰는 이 방법은 적합하지 않다.


## Service에서 `CompletableFuture.supplyAsync()` 사용

```java
@GetMapping("/something")
public CompletableFuture<ResponseEntity<ADto>> getSomething() {
    return this.aService.getSomething();
}

@Service
public CompletableFuture<ResponseEntity<ADto>> getSomething() {
    return CompletableFuture.supplyAsync(
        () -> ResponseEntity.ok(getSomethingTakesLong()),
        this.customTaskExecutor
    );
}
```

아무래도 서비스에서 `CompletableFuture`를 사용하는 것이 자연스러워 보인다. 다만 이 방식에서는 서비스가 `ResponseEntity`에 의존하게 되는 점이 상당히 부자연스럽다.


## Service에서 `CompletableFuture.supplyAsync()` 사용 + Controller에서 `CompletableFuture`의 `thenApply()` 처리

```java
@GetMapping("/something")
public CompletableFuture<ResponseEntity<ADto>> getSomething() {
    return this.aService.getSomething()
        .thenApply(ResponseEntity::ok);
}

@Service
public CompletableFuture<ADto> getSomething() {
    return CompletableFuture.supplyAsync(
        () -> getSomethingTakesLong(),
        this.customTaskExecutor
    );
}
```

서비스에서 `ResponseEntity`에 의존하지 않고 `CompletableFuture`를 반환하고, 컨트롤러에서는 서비스로부터 반환받은 `CompletableFuture`를 `thenApply()`로 체이닝하는 방식으로 의미상으로나 가독성에서 가장 낫다.

