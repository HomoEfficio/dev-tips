# Rust 에러 처리

Rust는 예외를 사용하지 않고, 복구 가능한 에러와 복구 불가능한 에러를 구분해서 처리한다. 복구 가능한 에러는 `Result<T, E>`로 처리하고, 복구 불가능한 에러는 `panic!`으로 처리한다. `panic!` 매크로가 호출되면 프로그램 실행이 종료된다.

## Result

Result는 다음과 같은 enum 이다.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

자바였다면 NumberFormatException 발생 지점에서 프로그램이 종료됐겠지만, Rust에서는 에러도 `Result`에 담겨진 채로 프로그램이 정상 종료된다.

```rust
fn main() {
    let num = "3".parse::<i32>();
    println!("{:?}", num);  // "{:?}" formatter를 쓰면 std::fmt::Debug에 정의한 출력방식으로 출력된다.
    
    let non_num = "a".parse::<i32>();
    println!("{:?}", non_num);
    
    let num = "5".parse::<i32>();  // rust는 같은 이름의 변수를 let으로 다시 선언 가능
    println!("{}", num.unwrap());  // unwrap()은 Result가 Ok이면 Ok안의 값을, Err이면 panic!()을 호출
}
//-----
Ok(3)
Err(ParseIntError { kind: InvalidDigit })
5
```

`unwrap()`은 편리하지만 `panic!()`을 호출해서 프로그램을 비정상 종료시킬 수 있으므로 실제 코드에서는 사용하지 않는 것이 좋다. 대신에 다음과 같이 `?`를 사용하는 것이 좋다. `?`는 에러가 아니면 `unwrap()`과 마찬가지로 Ok에 담긴 값을 반환하고 에러면 `return Err(From::from(err))`과 마찬가지로 `from()` 구현에 의해 반환되는 값을 Err에 담아서 바로 반환한다. `?` 한 글자만으로 `panic!()` 없이 바로 반환하므로 `map_or()` 등으로 chaining 하는 것보다 훨씬 간결하다. 다만 `?`를 사용하려면 `?`가 자동으로 호출하는 `From::from(err)`을 구현해야 한다.

