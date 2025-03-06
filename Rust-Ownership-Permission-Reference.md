# Rust Ownership and Reference

- Rust 공식 문서 https://rust-book.cs.brown.edu/ch04-00-understanding-ownership.html 내용 정리

## 기초 지식

### Safety

>Safety is the absence of undefined behavior.
>- 안전이란 정의되지 않은 행동이 없는 상태를 말한다.

- 그래서 어떤 포인터가 이미 해제된 메모리를 가리키고 있는 것만으로는 문제가 되지 않으며 safe 하다.
- 하지만 해제된 메모리를 사용하려 할 때 정의되지 않은 행동이 유발되며 unsafe 하게 된다.

>A foundational goal of Rust is to ensure that your programs never have undefined behavior.
>- Rust가 추구하는 가장 근본적인 목표는 프로그램에 정의되지 않은 행동이 없음을 보장하는 것이다.

>A secondary goal of Rust is to prevent undefined behavior at compile-time instead of run-time. 
>- 두 번째 목표는 정의되지 않은 행동이 없음을 런타임이 아니라 컴파일 타임에 보장하는 것이다.

### 안전성 보장을 위한 정책

>Rust Does Not Permit Manual Memory Management

>Box deallocation principle (fully correct): If a variable owns a box, when Rust deallocates the variable’s frame, then Rust deallocates the box’s heap memory.

### 스코프, 프레임, 스택

- https://www.perplexity.ai/search/in-rust-a-stack-frame-is-creat-6yWvL_lvQ_eeSppVBc6lnA
- 스코프는 스코프 안에 선언된 변수가 보여질 수 있는 범위를 나타낼뿐 메모리 할당, 해제와 직접적인 관련이 없다.
  ```rust
  fn main() { // 스택 프레임 생성
    let x = 10;
    {  // 스코프 생성
        let y = 20;
    } // 스코프 종료. y에 접근할 수 없게 되지만 이 시점에 y에 할당된 메모리까지 해제되는 것은 아니다.
    let z = 30;
  } // main 함수의 스택 프레임이 해제되며 이 시점에 z, y, x 순으로 해제된다.
  ```

- 프레임은 함수 당 하나씩 생성되며, 함수 실행이 종료되어 반환될 때 함수와 함께 생성된 프레임도 해제되며, 프레임 안에서 생성된 변수도 해제된다.
- 스택은 스레드 당 하나씩 생성된다.
- 메인 함수의 프레임은 메인 스레드 스택의 가장 아래에 생성된다.
- 다른 함수를 호출하면 호출된 함수의 프레임이 호출한 함수의 위에 쌓이고, 호출된 함수가 반환되면 호출된 함수의 프레임은 해제된다.
- 프레임 안에 있는 포인터 변수가 가리키는 힙 메모리 해제는 언어마다 다르다.
  - GC가 있는 언어는 사용자가 직접 해제하지 않아도 적정한 시점에 GC가 알아서 힙 메모리를 해제한다.
  - GC가 없는 언어는 사용자가 직접 해제하거나 언어가 특정 규칙에 따라 자동으로 해제해준다.
    - C는 사용자가 직접 해제해야 하고, Rust는 언어가 Ownership 같은 규칙에 따라 자동으로 해제해준다.
- 할당되지 않은 메모리를 참조해서 사용(읽기, 읽어서 수정, 삭제)하는 것을 허용하면 메모리 안전성이 깨진다.

## Ownership, Move

- 다음과 같이 함수 안에서 Box를 사용해서 힙 메모리 h를 할당 받고, h를 가리키는 포인터 p1, p2가 있다고 하자.

  ```rust
  fn main() {

    let p1 = Box::new(5);
    let p2 = p1;
  }
  ```

- 함수가 종료되면 p2가 먼저 메모리에서 해제되고 p2가 가리키는 힙 내 메모리 h도 해제된다.
- 이제 p1이 메모리에서 해제되면서 p1이 가리키는 힙 내 메모리 h도 해제돼야 하는데, h는 이미 해제됐으므로 다시 해제하기 위해 접근하면 메모리 안전성이 깨진다.
- 동일한 힙 내 메모리는 한 번만 해제돼야 안전하다.
- 메모리가 한 번만 해제되려면 p1, p2 중에 하나만 힙 내 메모리 해제에 관여해야 한다.
- 메모리 해제에 관여할 수 있는 자격을 소유권(Ownership)이라고 한다.
- 메모리 안전성을 보장하려면 특정 메모리에 대한 Ownership은 1개만 존재해야 한다.
- 따라서 Ownership을 가진 변수 p1이 다른 변수 p2에 할당되면,
  - p1이 가지고 있던 Ownership도 p2로 옮겨져야 1개의 Ownership을 보장할 수 있고,
    - 1개의 Ownership이 보장돼야 메모리 안전성을 보장할 수 있다.
- 프로그램의 안전성을 보장하기 위해 소유권이 없는 p1은 더 이상 사용할 수 없게 된다.

  ```rust
  fn main() {

      let p1 = Box::new(-5);
      let p2 = p1;
      my_func(p1);  // p1 사용 불가!
  }

  fn my_func(ib: Box<i32>) {
      
  }
  ```

- 컴파일러는 프레임 내에서 소유권이 명시적으로 이전(할당 또는 다른 함수의 파라미터로 전달)될 때만 프레임 종료 시 이전되기 전의 변수에 대해 메모리 해제를 하지 않는다.
  - 예를 들어 a 변수에 대해 프레임 내에서 다음과 같이 `*`를 사용해서 암시적 소유권 이전이 발생하면
    ```rust
    fn main() {
        let s = String::from("My String");
        let ref_s = &s;
        let s2 = *ref_s;  // 암시적 소유권 이전
        println!("{s2}");
    }  // 종료 시 같은 곳을 가리키는 s, s2 에 대해 double-free 발생
    ```
  - 프레임 종료 시 double-free unsafe behavior 가 발생한다.
  - 아래와 같이 다른 함수에 전달한 참조를 통해 다른 함수 내에서 `*`를 통한 암시적 소유권 이전에도 마찬가지 문제가 발행한다
    ```rust
    fn main() {
        let s = String::from("My String");
        implicit_move(&s);
    }

    fn implicit_move(param: &String) {
        let t = *param;  // cannot move out of `*param` which is behind a shared reference
    }
    ```


### Box deallocation principle

>Box deallocation principle (fully correct): If a variable owns a box, when Rust deallocates the variable’s frame, then Rust deallocates the box’s heap memory.

- Box는 Heap pointer 중 하나이므로, 위 내용 중 box를 pointer to a heap memory 라고 교체하고, 제목도 Heap pointer 해제 원칙이라고 하는 게 더 적합하다.
- 위 원칙에 따라 main 함수 종료 시점에서 힙 메모리 소유권을 가진 p2 에 의해 힙 메모리가 해제되며,
- p1은 소유권을 상실했으므로 메모리 해제에 관여하지 못한다.

### Moved heap data principle

>Moved heap data principle: if a variable x moves ownership of heap data to another variable y, then x cannot be used after the move.

- if 안에 move를 유발하는 코드가 있다면 실제 move 발생 여부는 런타임에서 결정되겠지만,
- Rust 컴파일러는 컴파일 타임에 엄격하게 안전을 보장하기 위해서 move 가 발생한다고 간주한다.

```rust
fn main() {

    let s = String::from("hello");
    let s2;
    let b = false;
    if b {
        s2 = s;
    }
    println!("{}", s);  // b 의 참/거짓과 무관하게 컴파일러는 move 가 발생한다고 간주하고 s 사용 시도 시 컴파일 에러를 발생한다.
}
```

## Reference, Borrow

- 참조(Reference)가 무엇인지 알아보기 전에 참조가 왜 필요한지부터 알아보자.

```rust
fn main() {
    let m1 = String::from("Maximum count exceeds!!!");

    send_slack(m1);
    send_email(m1);  // value used here after move
}

fn send_slack(msg: String) {
    println!("{} to slack", msg);
}

fn send_email(msg: String) {
    println!("{} to email", msg);
}

```

- 동일한 메시지를 슬랙과 이메일로 전송해야 할 때, 이미 `send_slack()`를 호출하면서 m1의 소유권이 이동돼서 `send_email()` 호출할 때는 m1을 사용할 수 없다.
- `let m2 = String::from("Maximum count exceeds!!!");`와 같이 동일한 문자열을 사용 횟수만큼 게속 만들어야 한다면 아무도 그런 불편한 언어를 사용하지 않을 것이다.
- 이 문제를 해결하기 위해 참조가 필요하다. 변수 앞에 `&`를 붙여서 참조를 만들어 사용하면 문자열을 여러번 사용할 수 있다.

```rust
fn main() {
    let m1 = String::from("Maximum count exceeds!!!");

    send_slack(&m1);
    send_email(&m1);
}

fn send_slack(msg: &String) {
    println!("{} to slack", msg);
}

fn send_email(msg: &String) {
    println!("{} to email", msg);
}
```

- 참조가 왜 필요한지 알았으니 이제 참조가 무엇인지 알아보자.

### References Are Non-Owning Pointers  

>- 참조는 소유권이 없는 포인터다

- 위 예제에서 `&m1`은 자산이 가리키는 `m1`에 대해서도, `m1`이 가리키는 문자열에 대해서도 아무런 소유권을 갖지 못한다.
- 그래서 `send_slack()` 함수 실행이 종료되면서 `msg`가 사라지더라도 `m1`과 `m1`이 가리키는 문자열은 메모리에서 해제되지 않는다.
- 즉, 다음과 같이 정리할 수 있다.
>참조는 소유권이 없기 때문에 메모리 해제에 관여하지 못하므로,  
>동일한 데이터를 가리키는 참조를 여러 개 만들어도 참조 단독으로는 메모리 관련 문제를 유발하지 않는다.

- 그리고 참조를 생성하는 것을 빌린다(borrow)라고 표현한다. 참조를 생성하면 참조가 가리키는 변수에 대한 소유권은 없지만, 변수를 사용(읽기와 쓰기)할 수는 있으므로, 빌리는 거라고 할 수 있다.

### Rust Avoids Simultaneous Aliasing and Mutation

- 참조 단독으로 메모리 관련 문제는 유발하지 않지만, 참조와 값 변경(Mutation)이 함께 엮이면 참조도 메모리 문제를 유발할 수 있다.

```rust
let mut v: Vec<i32> = vec![1, 2, 3];
let num: &i32 = &v[2];
v.push(4);
println!("Third element is {}", *num);
```

- `num`은 `v`의 세번째 원소에 대한 별명(Alias)이다.
- `v.push(4)`는 원소 4개 짜리 새로운 벡터 생성을 유발하므로 `v`도 새로운 메모리를 가리키게 되며, 기존의 3개 짜리 벡터는 메모리에서 해제된다.
- `num`은 메모리에서 해제된 기존의 3개짜리 벡터의 세번째 원소를 가리키므로, `num`을 사용할 때 unsafe behavior가 발생한다.
- Rust는 이 문제를 막기 위해 다음과 같은 원칙을 적용한다.

>Pointer Safety Principle: data should never be aliased and mutated at the same time.

## Permission, Lifetime

- 참조를 포함, 모든 변수는 읽기(Read), 쓰기(Write), 처분(Own), 이렇게 3가지 퍼미션(Permission, 권한)을 가지고 있다.
- 이중 Own 권한은 Ownership과 혼동하기 쉬운데,
  - Own 권한은 이름에는 소유라고 돼 있지만 소유권(Ownership)과는 무관하며,
  - Own 권한은 소유권에 대한 처분권이라고 이해하는 편이 낫다.
- 퍼미션은 컴파일 타임에 borrow checker 가 Pointer Safety Principle 준수를 확인하기 위해 사용하는 개념이라서 컴파일 타임에만 존재한다.
- 런타임에서는 퍼미션이라는 개념이 존재하지 않는다.
- Lifetime 은 퍼미션의 존속 기간을 말하며, 변수(참조 포함)의 최초 정의 시점부터 마지막으로 사용 완료된 곳까지를 의미한다.
- 반면에 Ownership은 변수의 마지막 사용 이후에도 프레임 소멸 시점(함수 종료)까지 계속 존재한다.
- **Lifetime 은 Ownership 존속 기간보다 길 수 없다.**
  - 따라서 다음과 같이 참조를 반환하며 함수 종료 후에도 참조가 계속 사용될 수 있으면 컴파일 에러가 발생한다.
    ```rust
    fn main() {
        let suppliedStr = string_supplier();
        println!("Supplied string: {}", suppliedStr);
    }

    fn string_supplier() -> &String {
        let s = String::from("A string");
        &s;  // 참조가 return 을 통해 함수 종료 후에도 존속하게 된다. 하지만 s는 함수 종료 후에는 소유권이 사라지므로, s의 참조인 &s 도 사용될 수 없어야 한다.
    }
    ```

## Ownership, Permission, Reference

![](https://images2.imgbox.com/7c/8b/G6z2AbEo_o.png)

- 처음 초기화 되는 변수는 Read, Own(소유권이라기보다는 처분권이라고 생각하자)를 가지며 `mut` 변수라면 Write도 가진다.
- R, O가 있는 변수에 대해서만 소유권의 이동이나 빌려주기가 가능하다.
  - 처분권인 O가 있는 변수에 대해서만 소유권의 이동이나 빌려주기가 가능하지만,
  - R 이 없으면 아예 읽을 수도 없으므로,
  - 결과적으로 R, O가 있는 변수에 대해서만 처분(소유권의 이동이나 빌려주기)이 가능하다.
- 변수의 소유권이 이동 되면 그 변수가 가지고 있던 모든 퍼미션이 사라진다.
- 변수의 참조가 생성(빌려주기)되면 그 변수가 가지고 있던 퍼미션이 일시적으로 제거되고, 참조의 마지막 사용 지점이 지나면(참조의 Lifetime이 끝나면) 변수의 퍼미션이 복구된다.
  - immutable reference(shared reference) 생성 시 변수의 W, O 퍼미션 일시적 제거, R은 그대로 남음
    - immutable reference 는 R, O를 가진다.
  - mutable reference(unique reference) 생성 시 변수의 R, W, O 퍼미션 모두 일시적 제거
    - mutable reference 는 R, W, O를 가진다.
  - reference 의 deref 는 O를 가지지 못한다.


## Ownership 에러 수정

- 결국 메모리 관련 에러는 결국 다음과 같이 분류 가능
  - 해제된 메모리를 참조
    - 변경에 따른 데이터 이동에 의해 해제된 기존 메모리 위치를 참조
    - 해제된 메모리에 대한 해제 재시도
  - 메모리가 해제되지 않음
- Ownership 에러는 '해제된 메모리를 참조'하는 상황에서 발생


### Ownership 종료 후에도 Reference 가 사용되는 에러

```rust
fn string_supplier() -> &String {
    let s = String::from("A string");
    &s;  // 참조가 return 을 통해 함수 종료 후에도 존속하게 된다. 하지만 s는 함수 종료 후에는 소유권이 사라지므로, s의 참조인 &s 도 사용될 수 없어야 한다.
}
```

1. Ownership을 늘리든가
  - Ownership 이동
    ```rust
    fn string_supplier() -> String {
        let s = String::from("A string");
        s
    }
    ```
2. Lifetime 을 늘리든가
  - `'static` 참조 반환
    ```rust
    fn string_supplier() -> &'static str {
        "A string"
    }
    ```
3. borrow-check를 런타임으로 미루든가
  - RC pointer(Reference Count Pointer) 사용
    ```rust
    fn string_supplier() -> Rc<String> {
        let s = Rc::new(String::from("A string"));
        Rc::clone(&s)
    }
    ```
4. 파라미터로 받은 `&mut` 에 넣어주든가
  - slot 사용
    ```rust
    fn string_supplier(output: &mut String) {
        output.replace_range(..,"A string");
    }
    ```

### immutable 파라미터에 대한 수정

```rust
fn stringify_name_with_title(name: &Vec<String>) -> String {
    name.push(String::from("Esq."));  // name은 immutable 인데 push 시도, 더 큰 데이터를 담을 수 있는 메모리 위치로 이동 발생
    let full = name.join(" ");
    full
}
```

- immutable 파라미터 수정을 허용하면 왜 unsafe 가 발생할까?
  - 참조되던 데이터의 메모리 상 위치가 다른 곳으로 변경될 수 있기 때문

```rust
fn main() {
    let name = vec![String::from("Ferris")];
    let first = &name[0];
    stringify_name_with_title(&name);  // 이 시점에서 first 가 가리키던 메모리는 해제된 상태
    println!("{}", first);  // 해제된 메모리를 가리키는 first 사용
}
```

1. `&mut`를 받도록 수정
  - immutable을 기대하는 함수 이름과 충돌
    ```rust
    fn stringify_name_with_title(name: &mut Vec<String>) -> String {
        name.push(String::from("Esq."));
        let full = name.join(" ");
        full
    }
    ```
2. `&Vec<String>` 대신에 `Vec<String>`을 받도록 수정
  - 호출하는 쪽 변수의 소유권이 소멸하므로 부적절
3. `clone` 사용
  - 메모리 과다 사용
    ```rust
    fn stringify_name_with_title(name: &Vec<String>) -> String {
        let mut name_clone = name.clone();
        name_clone.push(String::from("Esq."));
        let full = name_clone.join(" ");
        full
    }
    ```
4. `slice::join`이 내부적으로 값을 복사한다는 사실 이용
  - 가장 좋은 해법
    ```rust
    fn stringify_name_with_title(name: &Vec<String>) -> String {
        let mut full = name.join(" ");
        full.push_str(" Esq.");
        full
    }
    ```

### slice

- 메모리 상에서 인접해 있는 일정 범위의 데이터를 가리키는 특별한 레퍼런스
- 일반적인 레퍼런스는 스택에 있는 변수를 가리키는 반면, 슬라이스는 직접 힙에 있는 데이터를 가리킨다
- 따라서 슬라이스가 가리키는 데이터의 크기가 얼마인지 알아야 하며, 
  - 일반적인 레퍼런스와 다르게 크기라는 메타 정보를 포함하므로 fat pointer 라고 부르기도 한다.

![](https://images2.imgbox.com/f5/ca/M6STW0kr_o.png)


## 마무리

### 메모리 안전성

>- Rust 는 메모리 안전성을 보장하기 위해,
>- 즉 use-after-free 와 double-free 를 컴파일 타임에 방지하기 위해
>- Ownership 과 Permission 개념을 사용한다.
>    - Permission 은 컴파일 타임에만 존재하는 개념이다.
>- Reference 는 메모리 안전성 보장이라기보다는, 프로그래밍 편의를 위해 사용하는 개념이다.
>    - Reference 가 없다면 호출된 함수의 파라미터를 통해 변수의 소유권이 이전된 뒤에는 변수를 사용할 수 없게 된다.
>    - 호출된 함수 종료 이후에도 변수를 계속 사용려면 호출된 함수는 반드시 소유권을 반환하도록 프로그래밍해야 한다.

### Ownership, Permission, Reference

![](https://images2.imgbox.com/7c/8b/G6z2AbEo_o.png)

>- 처음 초기화 되는 변수는 Read, Own(소유권이라기보다는 처분권이라고 생각하자)를 가지며 `mut` 변수라면 Write도 가진다.
>- R, O가 있는 변수에 대해서만 소유권의 이동이나 빌려주기가 가능하다.
>    - 처분권인 O가 있는 변수에 대해서만 소유권의 이동이나 빌려주기가 가능하지만,
>    - R 이 없으면 아예 읽을 수도 없으므로,
>    - 결과적으로 R, O가 있는 변수에 대해서만 처분(소유권의 이동이나 빌려주기)이 가능하다.
>- 변수의 소유권이 이동 되면 그 변수가 가지고 있던 모든 퍼미션이 사라진다.
>- 변수의 참조가 생성(빌려주기)되면 그 변수가 가지고 있던 퍼미션이 일시적으로 제거되고, 참조의 마지막 사용 지점이 지나면(참조의 Lifetime이 끝나면) 변수의 퍼미션이 복구된다.
>    - immutable reference(shared reference) 생성 시 변수의 W, O 퍼미션 일시적 제거, R은 그대로 남음
>        - immutable reference 는 R, O를 가진다.
>    - mutable reference(unique reference) 생성 시 변수의 R, W, O 퍼미션 모두 일시적 제거
>        - mutable reference 는 R, W, O를 가진다.
>    - reference 의 deref 는 O를 가지지 못한다.

