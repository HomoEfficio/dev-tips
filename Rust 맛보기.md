# Rust 맛보기

## Fast and Safe

- GC나 별도의 Runtime이 없어 빠르다.
- Thread-safety가 언어 차원에서 컴파일 타임에 보장된다.
- 컴파일러가 해주는 게 많아서일까 컴파일 속도는 살짝 느린 감이 있다.
    
## Most Loved Language

- 2016, 2017, 2018, 2019 무려 4년 연속 스택오버플로우 설문 조사에서 가장 사랑받는 언어 1위!!
    - https://insights.stackoverflow.com/survey/2016/#technology-most-loved-dreaded-and-wanted
    - https://insights.stackoverflow.com/survey/2017/#most-loved-dreaded-and-wanted
    - https://insights.stackoverflow.com/survey/2018/#most-loved-dreaded-and-wanted
    - https://insights.stackoverflow.com/survey/2019#most-loved-dreaded-and-wanted
    
    
## 아직 살짝 부족한 Ecosystem

- IntelliJ에서는 아직 Debug 모드가 동작하지 않는다 ㅠㅜ (CLion에서는 동작한다고 한다)
- gRPC에서도 Rust는 지원 안함(구글은 Go를 밀테니 Rust는 지원할 계획도 없어 보임)
- ReactiveX에서도 마지막 커밋은 [2015년](https://github.com/ReactiveX/RxRust).. 
- 그래도 http://www.arewewebyet.org/ 이렇게 웹 프레임워크가 발전하고 있다.

## Rust도 요즘 언어들 있는 거 다 있다

- 패키지/빌드 시스템: https://doc.rust-lang.org/cargo/guide/
- 모듈 시스템: https://crates.io/
- Dev Kit 버전 관리: nvm 같은.. https://rustup.rs/
- 제네릭: https://doc.rust-lang.org/book/ch10-00-generics.html
- 람다식: https://doc.rust-lang.org/book/ch13-01-closures.html
    - lambda expression 대신 `closure`라는 이름으로 지원. 근데 생김새가 좀 이질적.. `|param| expression` 이렇게 생겼다..
- 강려크한 패턴 매칭: https://doc.rust-lang.org/book/ch18-00-patterns.html
- 코루틴(제너레이터): https://doc.rust-lang.org/std/ops/trait.Generator.html 아직 베타라고 한다.
- 문서화: https://doc.rust-lang.org/rustdoc/
- 웹 플레이그라운드: https://play.rust-lang.org/

## 눈물 없이는 볼 수 없는 감동적인 컴파일 에러 메시지

- 난...ㄱ ㅏ끔... 눈물을 흘린ㄷ ㅏ....  
![](http://img.etnews.com/news/article/2017/09/03/cms_temp_article_03185806264686.jpg)
- 아래 예제 코드에서 느낄 수 있다.


# GC 없는 메모리 관리

힙에 생성되지 않고 스택 내의 프레임에 생성되는 변수는 프레임이 종료되어 스택에서 빠져나갈 때 메모리가 함께 해제되므로 별도의 메모리 관리가 필요하지 않다.

하지만 힙에 생성되는 변수는 별도의 메모리 해제 과정이 필요하다. C/C++에서는 이 해제 과정을 개발자가 직접 해줘야 하고(C++은 스마트포인터로 이 부분을 해소한다), Java는 개발자가 하지 않고 Garbage Collector가 담당한다.

Rust에서는 개발자가 직접 하지도 않고, Garbage Collector도 존재하지 않는다. 대신에 **힙에 생성되는 변수를 참조하는 변수가 포함된 스코프가 종료되면 스코프에 있던 변수에 할당된 메모리도 해제되고, 그 변수가 참조하는 힙에 생성된 변수에 할당된 메모리도 함께 해제**된다.

>**Ownership을 가진 변수(Owner)는 자기가 포함된 스코프가 종료되면**,  
>**자기가 소유하던(참조하던) 변수들과 함께 메모리에서 해제(Drop)된다.**

## 변수에 대한 Ownership/Move 개념 도입

- **Ownership은 lifetime을 결정할 수 있는 권한**
- 힙에 생성되는 변수를 다른 변수에 할당하면 Ownership은 복사되지 않고(Not Copy) 이동(Move)된다. 즉, **Owner는 언제나 1개다.**
    - Ownership을 잃어버린 변수는 uninitialized 상태가 되며, 이 변수를 다시 초기화하지 않고 사용하면 컴파일 에러
    - 아래와 같이 힙에 문자열을 생성하는 `String::from()`을 통해 만든 문자열을 원소로 갖는 튜플의 Ownership은 복사되지 않고 이동되므로 아래와 같이 컴파일 에러가 발생한다.  
Rust의 컴파일 에러 메시지는 상당히 구체적이고 친절하다.
        ```rust
        fn main() {
            let a: i32 = 32;
            let b = a;  // 스택 안에서 값이 통째로 복사
            println!("a is {}", a);
            println!("b is {}", b);
            
            // String::from("Jordan")은 힙에 생성되므로
            let t1 = (23, String::from("Jordan"));
            // 아래와 같이 할당되면 t1의 Ownership은 t2로 이동되며, 
            let t2 = t1;
            // 이미 Ownership을 잃고 uninitialized 된 t1을 사용하려고 하면 컴파일 에러 발생
            let t1_1_len = t1.1.len();

            println!("t1_1_len() is {}", t1_1_len);
            println!("t2.1.len() is {}", t2.1.len());
        }

        //-----
        error[E0382]: borrow of moved value: `t1`
          --> src/main.rs:12:24
           |
        10 |         let t2 = t1;
           |                  -- value moved here
        11 |         // 이미 Ownership을 잃고 uninitialized 된 t1을 사용하려고 하면 컴파일 에러 발생
        12 |         let t1_1_len = t1.1.len();
           |                        ^^^^ value borrowed here after move
           |
           = note: move occurs because `t1` has type `(i32, std::string::String)`, which does not implement the `Copy` trait
        ```

- 힙에 생성되지 않는 타입을 Copy Type이라고 하며, Copy 타입의 변수를 다른 변수에 할당하면 값 자체가 통째로 복사된다. Copy 타입은 다음과 같다.
    - int, float, char, bool
    - int, float, char, bool를 원소로 가지는 Tuple이나 고정 크기 배열 변수
    - Copy 타입은 생성되지 않으므로 값이 통째로 복사되고 Ownership도 별개인 새 변수가 생긴다. 따라서 안전하고 그래서 정상 실행된다.
        ```rust
        fn main() {
            let a: i32 = 32;
            let b = a;  // 스택 안에서 값이 통째로 복사
            println!("a is {}", a);
            println!("b is {}", b);
            
            let t1 = (23, "Jordan");  // "Jordan"은 힙에 생성되지 않음
            let t2 = t1;  // 스택 안에서 값이 통째로 복사
            let t1_1_len = t1.1.len();  // t1 튜플의 두 번째(튜플 인덱스는 0부터 시작) 원소의 길이

            println!("t1_1_len() is {}", t1_1_len);
            println!("t2.1.len() is {}", t2.1.len());
        }
        ```
        
- Non Copy Type인 String이 포함된 튜플은 Non Copy Type이 된다.

    ```rust
    fn main() {
        let t1 = (23, String::from("Jordan"));  // String::from("Jordan")은 힙에 생성되는 Non Copy Type -> 이를 포함하는 튜플도 Non Copy Type
        let t2 = t1;  // t1이 Non Copy Type이므로 값 복사가 아니라 Ownership이 이전
        let t1_0 = t1.0;  // Ownership이 이전되어 Uninitialized된 t1 튜플을 참조하면서 컴파일 에러 발생

        println!("t1_0 is {}", t1_0);
        println!("t2 is {:?}", t2);
    }


       Compiling playground v0.0.1 (/playground)
    error[E0382]: use of moved value: `t1`
     --> src/main.rs:4:16
      |
    2 |     let t1 = (23, String::from("Jordan"));  // String::from("Jordan")은 힙에 생성되는 Non Copy Type -> 이를 포함하는 튜플도 Non Copy Type             ...
      |         -- move occurs because `t1` has type `(i32, String)`, which does not implement the `Copy` trait
    3 |     let t2 = t1;  // t1이 Non Copy Type이므로 값 복사가 아니라 Ownership이 이전
      |              -- value moved here
    4 |     let t1_0 = t1.0;  // Ownership이 없어 Uninitialized된 t1 튜플을 참조하면서 컴파일 에러 발생
      |                ^^^^ value used here after move

    For more information about this error, try `rustc --explain E0382`.
    error: could not compile `playground` due to previous error
    ```

- **다른 변수에 할당할 때뿐아니라 함수에 인자로 넘길 때도, 함수에서 값을 반환할 때도 Ownership이 넘어간다.** 그래서 아래와 같이 함수에 인자로 넘기면서 이미 Ownership을 잃은 변수 name1을 다시 사용하는 코드는 컴파일 에러가 발생한다.  
Rust의 컴파일 에러 메시지는 볼 수록 매력적이다.

    ```rust
    fn main() {
        let name1 = String::from("Rust");    
        my_print(name1);
        
        let name2 = name1;    
        my_print(name2);
    }

    fn my_print(name: String) {
        println!("{}", name);
    }

    //------
    error[E0382]: use of moved value: `name1`
     --> src/main.rs:6:17
      |
    4 |     my_print(name1);
      |              ----- value moved here
    5 |     
    6 |     let name2 = name1;
      |                 ^^^^^ value used here after move
      |
      = note: move occurs because `name1` has type `std::string::String`, which does not implement the `Copy` trait
    ```

그런데 이렇게 할당할 때마다, 함수에 넘겨주거나, 함수로부터 반환받을 때마다 Ownership이 이동되기만 한다면 프로그래밍이 사실 상 불가능할 것 같다. 그래서 Rust에서는 Ownership/Move 뿐아니라 Reference/Borrow 도 존재한다.


## Reference/Borrow

Rust에도 참조(Reference)가 있는데, 결국은 주소값이고 그래서 읽어오는 데는 제약이 없다는 점에서는 다른 언어의 참조와 같다. 하지만 값의 변경에는 확연하게 다른 차이를 보여주는 제약 사항이 있는데, 바로 이 제약 사항이 Rust의 Thread-Safety를 보장해주는 핵심 장치다.

이에 대한 설명이 [Rust 공식 책](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)에 있지만, 읽고나서 여러모로 테스트 해 본 결과 아쉽게도 책에서 설명하지 않은 부분이 하나 있는데, 바로 **참조의 사용 여부**다. ~아마도 사용되지 않는 참조가 코드에 존재하는 것 자체가 리팩터링 대상이고 결국에는 코드에서 제거되는 것이 바람직하므로, 사용 여부를 굳이 설명하지 않은 걸로 추측해본다.~ 이유는 참조뿐 아니라 변수의 사용 여부를 검사하는 기능은 Rust 2018 부터 도입되었는데, 공식 책은 아직 Rust 2018 기준이 아니기 때문이다. **컴파일러는 참조의 실제 사용 여부를 감안해서 제약 사항의 준수 여부를 컴파일 타임에 검사**한다. 다시 말하면 사용되지 않는 참조에 대해서는 제약 사항 위배가 있더라도 컴파일 에러가 발생하지 않는다.

Rust의 참조의 제약 사항은 다음과 같다.

>하나의 동일한 스코프 내에서,
>
>1. 어떤 변수에 대해 실제 사용되는 읽기 전용 참조는 여러 개 존재할 수 있다.
>2. 어떤 변수에 대해 실제 사용되는 변경 가능 참조는 단 한 개만 존재할 수 있다.
>3. 어떤 변수에 대해 실제 사용되는 변경 가능 참조와, 실제 사용되는 읽기 전용 참조는 동시에 존재할 수 없다.

1, 2는 굳이 부연 설명 없어도 대충 수긍할 수 있는데, 3은 살짝 애매하다. 왜 동시에 존재하면 안 될까?

왜냐하면 **변경에 의해 참조 위치가 바뀔 수 있기 때문**이다. 예를 들어 capacity가 4인 Vector에 현재 3개의 원소가 들어있는데 여기에 원소 2개를 더 추가하면 length는 5가 되고 capacity 값인 4를 초과하므로, 해당 Vector는 더 큰 capacity(예를 들면 8)를 가진 새로운 위치로 이동해야 5개 모두를 품을 수 있게 된다. 이렇게 되면 읽기 전용 참조에 들어있던 주소값은 이동된 새로운 Vector를 가리키지 않으므로 유효하지 않은 참조가 되며 이런 상황을 막기 위해 3번 제약이 필요하다.

다음과 같이 실제 사용되는 변경 가능 참조와 실제 사용되는 읽기 전용 참조가 동시에 존재하면 컴파일 에러가 발생한다. 컴파일 에러 발생 위치도 눈여겨 보자.

```rust
fn main() {
    let mut str = String::from("Rust");

    let r1_str = &str;

    let w1_str = &mut str;
    
    println!("{}", r1_str);
    
    println!("{}", w1_str);
}

//-----
error[E0502]: cannot borrow `str` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:18
  |
4 |     let r1_str = &str;
  |                  ---- immutable borrow occurs here
5 | 
6 |     let w1_str = &mut str;
  |                  ^^^^^^^^ mutable borrow occurs here
7 |     
8 |     println!("{}", r1_str);
  |                    ------ immutable borrow later used here
```

나중에 초기화한 변경 가능 참조 초기화 부분에서 컴파일 에러가 발생했다.

그럼 초기화 순서를 바꾸면 어떨까?

```rust
fn main() {
    let mut str = String::from("Rust");

    let w1_str = &mut str;
    
    let r1_str = &str;

    println!("{}", r1_str);
    
    println!("{}", w1_str);
}

//-----
error[E0502]: cannot borrow `str` as immutable because it is also borrowed as mutable
  --> src/main.rs:6:18
   |
4  |     let w1_str = &mut str;
   |                  -------- mutable borrow occurs here
5  |     
6  |     let r1_str = &str;
   |                  ^^^^ immutable borrow occurs here
...
10 |     println!("{}", w1_str);
   |                    ------ mutable borrow later used here
```

마찬가지로 둘 중 나중에 초기화 되는 쪽에서 컴파일 에러가 발생한다. 초기화 순서 말고 참조가 실제 사용되는 순서를 바꿔봐도 둘 중 나중에 초기화 되는 쪽에서 컴파일 에러가 발생한다.

실제 사용되는 코드를 제거해보면 앞에서 살펴본 것과는 다르게 동작하는 것을 확인할 수 있다. 사용되지 않는 참조가 포함되어 있는 것은 어차피 현실에서는 있을 수 없는, 있어서는 안 되는 상황이라고 간주하고 여기에서 따로 설명하지 않겠지만, https://play.rust-lang.org/ 에서 따로 실험해보면 컴파일러가 참조의 실제 사용 여부를 감안한다는 것을 확인할 수 있을 것이다.

참조는 Ownership을 가지지 않으므로 참조가 스코프에서 사라진다고 해도 Owner가 여전히 살아있다면 참조가 가리키던 값(이게 결국 Owner) 역시 살아있다.

읽기 전용 참조는 Shared Reference라고 하며 Copy 타입이다. 따라서 할당, 인자로 넘기기 등에서 동일한 대상에 대한 참조가 여러 갈래로 복사되고 공유되어 사용된다.  
변경 가능 참조는 Mutable Reference라고 하며 Copy 타입이 아니다. 따라서 할당, 인자로 넘기기 등에 사용되면 이 참조가 가리키는 대상은 오직 이 참조를 통해서만 접근 가능하다. 

참조는 `.` 연산자, 참조의 참조, 참조의 비교 등 다양한 케이스에 대해 살펴볼 필요가 있는데 내용이 꽤 많으므로 별도로 다루기로 하고, 맛보기에서는 여기까지만 살펴보기로 한다.


## Rc, Arc

Rust에서는 **Owner는 단 하나** 원칙을 지키기 위해 Copy 타입이 아닌 변수는 할당, 인자 전달, 반환에서 복사가 아니라 Ownership이 이동된다. 하지만 필요하다면 `Rc<T>`, `Arc<T>`를 사용해서, 이동이 아니라 파이썬처럼 Reference count를 통해 다수의 Owner를 허용할 수도 있다. `Arc`는 Atomic Reference Count이며 Thread-safe를 보장한다.

다음 코드에서 t, u는 `clone()`을 통해 생성되지만 인스턴스 자체가 복제되는 게 아니라 `Rc<string>`에 대한 참조만 복제된다. 따라서 s, t, u는 모두 힙에 생성된 동일한 `Rc<string>` 타입 인스턴스를 가리키며, 이 인스턴스의 Reference count는 3이 되고, 0이 되면 이 인스턴스는 메모리에서 사라진다.

```rust
let s = Rc<string>::new("Oh".to_string());
let t = s.clone();
let u = s.clone();
```

**Owner가 여럿이라면 변경이 이슈가 될텐데, `Rc<T>`는 기본적으로 immutable 이라서 변경으로 인한 이슈를 예방**한다. 다만 Interior Mutability 라는 방법을 통해 mutable한 `Rc<T>`도 만들 수 있다고 한다.


## Lifetime parameter

lifetime은 어떤 참조가 안전하게 사용될 수 있는 구간 또는 범위를 의미한다. lexical 블록, 하나의 문(statement), 하나의 식(expression), 변수의 스코프 등이 바로 그 구간 또는 범위가 될 수 있다. lifetime은 컴파일 타임에만 사용되는 개념이며 런타임에는 존재하지 않는다. 출처: Programming Rust 5장

결국 lifetime은 안전하지 않은 참조를 사용하는 코드를 작성하지 않도록 미리 막아주는 장치인데, 여기서 안전하지 않은 참조라는 것은 크게 두 가지 관점에서 살펴볼 수 있다.

```rust
fn main() {
    let out_live_ref;
    {
        let short_lived_int = 3;
        out_live_ref = &short_lived_int;  // 1. `&short_lived_int`는 자신이 할당된 out_live_ref 보다 일찍 소멸한다.
    }
    println!("{}", *out_live_ref);  // 2. `out_live_ref`는 Owner인 `short_lived_int`가 소멸한 후에 사용되고 있다.
}
```

위 코드에서 `short_lived_int`는 Owner이고, `&short_lived_int`와 `out_live_ref`는 참조다.
참조가 안전하게 사용된다는 것은 아래의 2가지 제약사항을 모두 준수한다는 것을 의미한다.  
- 주석 1에 나타난 것처럼, **참조(여기서는 `&short_lived_int`)는 적어도 참조 자신이 할당된 변수(`out_live_ref`)의 생존 기간만큼 살아남아야 한다. (최소 한도)**
- 주석 2에 나타난 것처럼, **참조(여기서는 `out_live_ref`)는 길어도 참조 자신이 가리키는 값의 Owner(`short_lived_int`)의 생존 기간보다 오래 살아남으면 안 된다. (최대 한도)**

위 코드는 위 2가지 사항을 모두 만족하지 못하므로 안전하지 못한 참조를 사용하고 있으며 따라서 컴파일 에러가 발생한다. 두 가지 관점에서 살펴봤지만 결론은 사실 한 가지다. `out_live_ref`가 dangling reference가 된다는 것이다. 결국 **lifetime은 dangling reference가 발생하지 않게 해주는 장치**라고 할 수 있다. 그리고 **참조를 사용하는 함수, 메서드, struct, trait 등의 lifetime을 컴파일러에 알려줘야 컴파일러가 컴파일 타임에 dangling reference 발생 여부를 판단할 수 있게 된다. lifetime을 컴파일러에 명시적으로 알려주는 것이 바로 lifetime parameter다.**

따라서 **함수(또는 메서드, struct, trait 등)의 정의에 lifetime 파라미터가 붙어있고 컴파일이 성공되었다면, 그 함수(또는 메서드, struct, trait 등)에서는 dangling reference가 발생하지 않음을 컴파일러가 보장해준다는 것이 lifetime 파라미터의 효용**이라고 볼 수 있다.

lifetime parameter도 **제네릭 타입 파라미터처럼 꺽쇠를 써서 `<'a>`와 같은 형식으로 표기**되는데, 제네릭으로 타입을 알려주는 것이 아니라 **제네릭으로 lifetime을 알려주는 것이다.**

lifetime parameter는 C, C++, Java 등 다른 언어에 없는 개념이라 금방 와닿지가 않는다. 먼저 함수에 명시하는 lifetime parameter를 통해 감을 잡아보자.

### 함수에 명시하는 lifetime parameter

다음과 같은 함수 정의에서 

```rust
fn my_fun<'a>(r: &'a i32) -> &'a i32 {
    ...
}
```

- `'a`(tick A라고 읽는다)는 함수 my_fun의 lifetime parameter라고 한다.
- `r`은 임의의 lifetime `a`를 갖는 i32형 참조를 나타낸다.
- 반환되는 참조의 lifetime도 input 파라미터의 lifetime과 동일하다.


다음과 같은 코드는 컴파일에 성공한다.

```rust
static STATIC: &i32 = &222;

fn my_fun<'a>(r: &'a i32) -> &'a i32 {
    if *r > 5 {
        r
    } else {
        &5
    }
}

fn main() {
    let result;
    {
        result = my_fun(STATIC);
    }
    println!("result: {}", result);
}
```

함수 my_fun의 인자로 static 참조인 `STATIC` 참조를 넘기면 `'a`가 실질적으로는 `'static`이므로 `result`에는 static 참조가 할당되고 따라서 my_fun()이 호출되는 블록이 종료된 후에도 `result`는 dangling reference가 아니며 안전한 값을 가지고 있다.

하지만 다음과 같이 lifetime이 `'static`이 아닌 참조를 인자로 넘기면, 반환값의 lifetime도 `'static`이 아니라 인자로 받은 참조와 동일한 lifetime을 갖게 된다.  
my_fun()의 인자인 `&input`이 가리키는 값의 Owner인 `input`을 감싸고 있는 lexicl 블록이 종료되면,
- `input`이 소멸되고
- my_fun()의 반환값을 `rv`라고 할 때, my_fun()이 인자로 받은 `&input`과 동일한 lifetime을 갖는 `rv`도 사용될 수 없게 되므로,
- `rv`를 할당받은 `result`는 lexical 블록 종료 이후에는 dangling reference가 되며 dangling reference인 `result`가 사용되면 컴파일 에러가 발생한다.

```rust
fn my_fun<'a>(r: &'a i32) -> &'a i32 {
    if *r > 5 {
        r
    } else {
        &5
    }
}

fn main() {
    let result;
    {
        let input = 1;
        result = my_fun(&input);        
    }
    println!("result: {}", result);
}

//-----
error[E0597]: `input` does not live long enough
  --> src/main.rs:13:25
   |
13 |         result = my_fun(&input);
   |                         ^^^^^^ borrowed value does not live long enough
...
18 |     }
   |     - `input` dropped here while still borrowed
19 |     println!("result: {}", result);
   |                            ------ borrow later used here
```

위에서 살펴본 것처럼 **`'a`는 암시적으로 `'static` 일 수도 있다.**

하지만 명시적으로 `'static` 일 수는 없다. 다음과 같이 lifetime이 `'static` 인 static 참조를 인자를 넘기더라도, 파라미터의 lifetime은 `'a`이므로 **`'a`인 참조를 명시적으로 static mut 참조에 할당하면 컴파일 에러가 발생**한다. 

```rust
static mut GLOBAL: &i32 = &111;
static STATIC: &i32 = &222;

fn my_fun<'a>(r: &'a i32) {
    GLOBAL = r;
}

fn main() {
    my_fun(STATIC);
}

//-----
error[E0312]: lifetime of reference outlives lifetime of borrowed content...
 --> src/main.rs:5:14
  |
5 |     GLOBAL = r;
  |              ^
  |
  = note: ...the reference is valid for the static lifetime...
note: ...but the borrowed content is only valid for the lifetime 'a as defined on the function body at 4:11
 --> src/main.rs:4:11
  |
4 | fn my_fun<'a>(r: &'a i32) {
  |           ^^
```


## Slice

`Vec` 같은 컬렉션이나 문자열의 일부분에 대한 읽기 전용 참조를 Slice라고 한다.

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    
    println!("{:?}", &nums[0..3]);
}
//-----
[1, 2, 3]
```

위와 같이 숫자를 원소로 하는 Slice의 타입은 `&[i32]`, `&[f64]` 등으로 표현한다.


```rust
fn main() {
    let hello = String::from("Hello");
    
    println!("{}", &hello[1..4]);
}
//-----
ell
```

위와 같이 String의 Slice의 타입은 `&str`이며, `"Hello"`와 같은 문자열 리터럴은 사실은 `&str` 타입이다.


# 불변성

Rust에서는 모든 변수가 기본적으로 immutable이다. 그래서 값을 변경할 필요가 있는 변수에는 명시적으로 `mut`라는 키워드를 앞에 붙여줘야 한다. 반면에 Java에서는 기본이 mutable이고 불변성을 적용하려면 `final` 키워드를 붙여줘야 한다.

게다가 Java에서는 참조형 변수에 대해 `final`이라는 키워드를 붙여줘도, 그 변수에 다른 참조값을 할당할 수 없을 뿐 참조가 가리키는 객체는 여전히 mutable이다.(Java9 부터 ImmutableCollections가 추가됨)

```java
public void immutable__test() {
    final List<Integer> integers = new ArrayList<>();  // 이후 intergers에 다른 값 할당 불가
    
    integers.add(23);  // 참조가 가리키는 list에 23 추가는 가능

    assertThat(integers.size()).isEqualTo(1);  // true
}
```

하지만 Rust에서는 `mut`를 붙이지 않으면 참조형 변수가 가리키는 데이터구조도 불변이다.

```rust
fn main() {
    let integers = Vec::new();
    
    integers.push(1);
    
    println!("size of integers: {}", integers.len());
}

//-----
error[E0596]: cannot borrow `integers` as mutable, as it is not declared as mutable
 --> src/main.rs:4:5
  |
2 |     let integers = Vec::new();
  |         -------- help: consider changing this to be mutable: `mut integers`
3 |     
4 |     integers.push(1);
  |     ^^^^^^^^ cannot borrow as mutable
```

`integers` 대신에 `mut integers`를 써보는 건 어떻겠냐는 천사같은 컴파일 에러 메시지가 나온다.


# struct

TODO 구조화 된 데이터


# Closure

다른 언어에서는 주로 람다식(Lambda Expression) 또는 람다라고 부르는 것을 Rust에서는 클로저(Closure)라고 부른다.

람다식이나 클로저에서는 상위 스코프에 있는 변수에 접근해서 사용할 수 있고, 이렇게 사용되는 변수를 자유 변수라고 한다. GC를 사용하는 다른 언어에서는 자유 변수를 힙에 생성해서 상위 스코프가 종료되고 스택이 사라지더라도 람다식은 힙에 있는 자유 변수를 사용할 수 있다. 이렇게 상위 스코프의 스택에 있던 자유 변수를 상위 스코프의 스택이 사라지더라도 사용할 수 있게 되는 메커니즘을 capture라고 한다.

Rust에는 GC가 없으며, 자유 변수를 힙에 생성하지 않는다. 그럼 Rust는 자유 변수를 어떻게 capture해서 사용할 수 있을까?

Rust에는 앞에서 살펴본 것처럼 변수의 라이프사이클에 소유권 개념이 있다. 상위 스코프에서 클로저를 사용할 때 `move` 키워드를 사용해서 자유 변수의 소유권을 클로저에 명시적으로 넘겨주면, 클로저는 자유 변수를 자기 스택으로 가져온다. 자유 변수가 Copy 타입이면 값을 자기 스택으로 복사해오고, non-Copy 타입이면 힙에 생성된 자료 구조를 가리키던 non-Copy 값을 자기 스택으로 가져와서, 상위 스코프가 종료되어 스택이 사라지더라도 클로저는 자기 스택에 보유하고 있는 자유 변수를 통해 힙에 생성된 자료 구조를 사용할 수 있다. 코드는 다음과 같다.

```rust
fn main() {
    let hello = String::from("  Hello  ");
    
    {
        let trimmer = move |param: String| -> String { param.trim().to_owned() };
        
        println!("Trimmed by trimmer closure: [{:?}]", trimmer(hello));
    }
}


//-----
Trimmed by trimmer closure: ["Hello"]
```

아래와 같이 hello 를 다시 사용하면 컴파일 에러가 발생한다.

```rust
fn main() {
    let hello = String::from("  Hello  ");
    
    {
        let trimmer = move |param: String| -> String { param.trim().to_owned() };
        
        println!("Trimmed by trimmer closure: [{:?}]", trimmer(hello));
    }
    
    println!("hello: {:?}", hello);  // 여기!! hello 를 다시 사용 시도
}

//-----
   Compiling playground v0.0.1 (/playground)
error[E0382]: borrow of moved value: `hello`
  --> src/main.rs:10:29
   |
2  |     let hello = String::from("  Hello  ");
   |         ----- move occurs because `hello` has type `String`, which does not implement the `Copy` trait
...
7  |         println!("Trimmed by trimmer closure: [{:?}]", trimmer(hello));
   |                                                                ----- value moved here
...
10 |     println!("hello: {:?}", hello);
   |                             ^^^^^ value borrowed here after move
   |
   = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0382`.
error: could not compile `playground` due to previous error
```

따라서 자유 변수는 공유되지 않으며 Rust가 자랑하는 안전성을 계속 유지할 수 있다.


## 함수의 타입과 클로저의 타입

Rust에서는 함수도 값으로 취급될 수 있고, 클로저도 값으로 취급될 수 있다. 타입 기반 언어인 Rust에서 값으로 취급될 수 있다는 것은 타입을 가지고 있다는 말이다.

함수의 타입은 파라미터와 반환값으로 정해지며, 다음과 같은 함수의 타입은 `fn(&MyStruct) -> usize`이다.

```rust
fn show_respect(my_struct: &MyStruct) -> usize {
  my_struct.size
}
```

위 함수는 다음과 같은 함수의 인자로 사용될 수 있다.

```rust
fn higher_order_fn_1(my_struct: &MyStruct, test_fn: fn(&MyStruct) -> usize) -> bool {
  test_fn(my_struct) > 100
}

let result1 = higher_order_fn_1(my_struct, show_respect);
```

클로저의 타입도 파라미터와 반환값으로 정해지지만, 함수의 타입과 다르다. 그래서 다음과 같이 사용하면 타입 에러가 발생한다.

```rust
let result2 = higher_order_fn_1(my_struct, |param| param.size);
```

클로저는 `Fn`이라는 trait을 구현한다(`FnOnce`와 `FnMut`도 있다). 그리고 `Fn`는 함수를 인자로 받을 수 있다(즉, 함수도 `Fn` trait을 구현한다고 유추할 수 있다). 그래서 다음과 같이 파라미터의 타입을 바꾸면,

```rust
fn higher_order_fn_1<F>(my_struct: &MyStruct, test_fn: F) -> bool
    where F: Fn(&MyStruct) -> bool {
    test_fn(my_struct) > 100
}
```

다음과 같이 함수와 클로저 모두 인자로 넘길 수 있다.

```rust
let result1 = higher_order_fn_1(my_struct, show_respect);
let result2 = higher_order_fn_1(my_struct, |param| param.size);
```

인자의 소유권을 가져와서 사용하고 해당 인자를 다시는 사용할 수 없게 만드는 클로저는 `FnOnce` 타입이며, 2번 이상 호출될 수 없다.

인자의 참조를 빌려와서 사용하는 클로저는 `Fn` 타입이며, 2번 이상 호출될 수 있다.

인자의 참조를 mutable로 빌려와서 사용하는 클로저는 `FnMut` 타입이며, 값을 변경하면서 2번 이상 호출될 수 있다.



## 클로저 성능

다른 언어에서는 클로저가 힙에 할당되고, 동적 디스패치되고, 가비지 컬렉션되므로 성능 손실이 발생한다. 그리고 인라인화 되지 못 하므로 최적화 포인트도 적다.

Rust의 클로저는 가비지 컬렉션되지 않으며, Box나 Vec 또는 다른 컨테이너에 넣지 않는한 힙에 생성되지 않고, 타입이 있으므로 인라인화 될 수 있어서 다른 언어에서의 클로저가 유발하는 성능 손실이 없다.


# module, crate, workspace

- module은 프로젝트 내에서 코드를 조직화하는 단위(Java의 package)
- crate(상자라는 뜻)는 서로 다른 프로젝트 사이에서 코드를 재사용할 수 있는 단위
    - src와 target을 각자 따로 가지고 있음
- workspace는 여러 개의 crate를 담을 수 있는 하나의 프로젝트에 해당
    - crate는 기본적으로 target을 각자 따로 가지며 동일한 의존 라이브러리도 crate 마다 중복으로 컴파일되고 중복으로 존재
    - 하나의 workspace에 하나의 target만을 두고 workspace에 담긴 여러 crate가 사용하는 동일한 의존 라이브러리 공유 가능
    - toml 파일에 다음과 같이 명시하고,

        ```toml
        [workspace]
        members = ["crate01", "crate02", "crate03"]
        ```

    - 각 crate 폴더 아래에 있던 cargo.lock 파일과 target 디렉터리를 지우면, workspace 루트 바로 아래에 전체 workspace를 아우르는 cargo.lock 파일과 target 디렉터리가 생성된다.


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.






