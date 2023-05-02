# Rust의 참조

## `.` 연산자

Rust는 매우 딱딱하고 보이는 그대로 동작하지만, 생산성이나 가독성을 위해 `.` 연산자에는 유연성을 부여했다.

### dereference

`.` 앞의 변수인 t가 참조일 경우 `t.`는 `(*t).`와 같다. 즉, 암묵적으로 역참조 연산을 수행한다.

```rust
fn main() {
    let mut s = "Homo Efficio".to_string();
    let t = &mut s;
    t.push('!');
    (*t).push('*');
    println!("{}", t);
}

//-----
Homo Efficio!*
```

### borrow

`.` 앞의 변수 t가 참조가 아니면 `t.a`는,  
- `a`가 immutable 이면 `(&t)`와 같고,  
- `a`가 mutable 이면 `(&mut t).`와 같다. 

이 사실은 Rust 컴파일러의 친절한 오류 메시지를 통해 더 명백하게 확인할 수 있다.

1. immutable borrow

    ```rust
    fn main() {
        let mut s = "homo.efficio".to_string();
        let t = &mut s;
        println!("{}", s.len());
        println!("{}", t.len());
    }

    //-----
    error[E0502]: cannot borrow `s` as immutable because it is also borrowed as mutable
     --> src/main.rs:4:20
      |
    3 |     let t = &mut s;
      |             ------ mutable borrow occurs here
    4 |     println!("{}", s.len());
      |                    ^ immutable borrow occurs here
    5 |     println!("{}", t.len());
      |                    - mutable borrow later used here
    ```

    `s`는 참조가 아닌데, 4행의 `s.len()`에서 immutable borrow 가 발생했다고 나온다. 결국 `s.len()`라고 작성했지만 내부적으로는 `(&s).len()`으로 동작함을 알 수 있다.

1. mutable borrow

    ```rust
    fn main() {
        let mut s = "homo.efficio".to_string();
        let t = &s;
        s.push('!');
        println!("{}", t);
    }

    //-----
    error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
     --> src/main.rs:4:5
      |
    3 |     let t = &s;
      |             -- immutable borrow occurs here
    4 |     s.push('!');
      |     ^^^^^^^^^^^ mutable borrow occurs here
    5 |     println!("{}", t);
      |                    - immutable borrow later used here
    ```

    `s`는 mutable이고 참조가 아닌데, 4행의 `s.push('!')`에서 mutable borrow 가 발생했다고 나온다. 결국 `s.push('!')`라고 작성했지만 내부적으로는 `(&mut s).push('!')`으로 동작함을 알 수 있다.

    참고로 위 코드에서 빌려온 참조인 `t`를 사용하지 않으면 컴파일 에러는 발생하지 않고, `t`가 사용되지 않는다는 경고만 나온다.

    ```rust
    fn main() {
        let mut s = "homo.efficio".to_string();
        let t = &s;
        s.push('!');
        // println!("{}", t);
    }

    //-----
    warning: unused variable: `t`
     --> src/main.rs:3:9
      |
    3 |     let t = &s;
      |         ^ help: consider using `_t` instead
      |
      = note: #[warn(unused_variables)] on by default
    ```

## loop

함수에 값을 넘기면서 Ownership을 이동시킬 때와 참조를 넘기면서 빌려줄 때, 받는 함수 안에서의 loop도 다르게 동작한다.

```rust
use std::collections::HashMap;

type Table = HashMap<String, Vec<String>>;

fn main() {
    let mut table = Table::new();
    table.insert("omwomw".to_string(), vec!["abc.com".to_string(), "def.com".to_string()]);
    table.insert("hanmomhanda".to_string(), vec!["gmail.com".to_string(), "facebook.com".to_string()]);
    table.insert("homo.efficio".to_string(), vec!["github.io".to_string()]);
    show(table);
}

fn show(table: Table) {
    for (name, domains) in table {
        println!("Domains of {}", name);
        for domain in domains {
            println!("  {}", domain);
        }
    }
    // table의 ownership이 for in 루프로 이동되므로 
    // 여기에서 table을 다시 사용하면 컴파일 에러 발생
    println!("size of table: {}", table.len());  
}

//-----
error[E0382]: borrow of moved value: `table`
  --> src/main.rs:22:35
   |
14 |     for (name, domains) in table {
   |                            ----- value moved here
...
22 |     println!("size of table: {}", table.len());
   |                                   ^^^^^ value borrowed here after move
   |
   = note: move occurs because `table` has type `std::collections::HashMap<std::string::String, std::vec::Vec<std::string::String>>`, which does not implement the `Copy` trait
```

`show()`에 `table`을 직접 값으로 넘기면 `for (name, domains) in table`의 `in table`에서도 `table`의 ownership이 반복문으로 이동되고, `for (name, domains)`에서도 HashMap의 한 원소의 ownership이 `name`, `domains`으로 이동된다. 따라서 그 이후에 `table.len()` 를 사용하면, 앞의 `.` 연산자에서 살펴본 것처럼 내부적으로 borrow가 작동하면서 위와 같이 컴파일 에러가 발생한다.

하지만, 직접 값이 아니라 참조인 `&table`로 `show()`에 넘기면 `table`의 타입도 `&Table`이 되고, `name`, `domains`의 타입도 `&String`, `&Vec<String>`이 되며, 빌려온 참조이므로 `table`을 사용할 수 있다.

```rust
use std::collections::HashMap;

type Table = HashMap<String, Vec<String>>;

fn main() {
    let mut table = Table::new();
    table.insert("omwomw".to_string(), vec!["abc.com".to_string(), "def.com".to_string()]);
    table.insert("hanmomhanda".to_string(), vec!["gmail.com".to_string(), "facebook.com".to_string()]);
    table.insert("homo.efficio".to_string(), vec!["github.io".to_string()]);
    show(&table);
}

fn show(table: &Table) {
    for (name, domains) in table {
        println!("Domains of {}", name);
        for domain in domains {
            println!("  {}", domain);
        }
    }

    println!("size of table: {}", table.len());
}

//-----
Domains of homo.efficio
  github.io
Domains of omwomw
  abc.com
  def.com
Domains of hanmomhanda
  gmail.com
  facebook.com
size of table: 3
```

## 참조의 참조

Rust에서는 참조의 참조, 참조의 참조의 참조의.. 가 가능하다. 그리고 아래 코드처럼 참조의 참조의 참조를 출력하면 원래의 값이 출력된다. 물론 Rust에서도 타입 추론을 지원하므로 t, u, v 우측에 있는 타입은 실제로는 명시하지 않아도 된다. 

```rust
fn main() {
    let mut s = "Homo Efficio".to_string();
    let mut t: &mut String = &mut s;
    let mut u: &mut &mut String = &mut t;
    let v: &mut &mut &mut String = &mut u;
    
    println!("{}", v);
}

//-----
Homo Efficio
```

### 무결성 유지

위 코드를 보면 mutable reference가 여러 개 생성되므로 t, u, v 각각을 통해 값을 바꿀 수 있게 되는데, 이는 Rust가 자랑하는 코드 안정성이 무너지는 것이 아닌가 하는 생각이 들 수 있는데 맞는 얘기다. 위 코드는 v가 읽기 전용, 그러니까 shared reference로 사용되면서 u, v 모두 shared reference로 사용되기 때문에 컴파일 에러가 발생하지 않은 것이다.

다음과 같이 mutable로 사용되면,

```rust
fn main() {
    let mut s = "Homo Efficio".to_string();
    let mut t: &mut String = &mut s;
    let mut u: &mut &mut String = &mut t;
    let v: &mut &mut &mut String = &mut u;
    t.push('!');
    println!("{}", v);
}

//-----
error[E0499]: cannot borrow `*t` as mutable more than once at a time
 --> src/main.rs:6:5
  |
4 |     let mut u: &mut &mut String = &mut t;
  |                                   ------ first mutable borrow occurs here
5 |     let v: &mut &mut &mut String = &mut u;
6 |     t.push('!');
  |     ^ second mutable borrow occurs here
7 |     println!("{}", v);
  |                    - first borrow later used here
```

똑똑한 컴파일러가 위와 같이 mutable로 여러 번 빌려줄 수 없다고 친절하게 알려준다.

### 비교 연산

비교 연산자를 통해 참조를 비교하면 참조가 최종적으로 가리키는 값을 대상으로 비교한다. 다음 코드를 보면 확인할 수 있다.

```rust
fn main() {
    let a = 10;
    let b = 10;

    let aa = &a;
    let bb = &b;

    let aaa = &aa;
    let bbb = &bb;

    assert!(aa == bb);
    assert!(aaa == bbb);

    assert!(aa != bb);  // 여기서 panick 발생
}

//-----
thread 'main' panicked at 'assertion failed: aa != bb', src/main.rs:14:5
```




