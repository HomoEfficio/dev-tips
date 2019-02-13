# Rust 맛보기

## Fast and Safe

- GC나 별도의 Runtime이 없어 빠르다.
- Thread-safety가 언어 차원에서 컴파일 타임에 보장된다.
    - 컴파일러가 해주는 게 많아서일까 컴파일 속도는 살짝 느린 감이 있다.
    
## Most Loved Language

- 2016, 2017, 2018 이렇게 3년 연속 스택오버플로우 설문 조사에서 가장 사랑받는 언어 1위!!
    - https://insights.stackoverflow.com/survey/2016/#technology-most-loved-dreaded-and-wanted
    - https://insights.stackoverflow.com/survey/2017/#most-loved-dreaded-and-wanted
    - https://insights.stackoverflow.com/survey/2018/#most-loved-dreaded-and-wanted
    - 이게 매년 3월에 발표되는데 2019년은 어떨지.. (아마 코틀린이 1위 할 것 같기도..)
    
## 아직 살짝 부족한 Ecosystem

- IntelliJ에서는 아직 Debug 모드가 동작하지 않는다 ㅠㅜ (CLion에서는 동작한다고 한다)
- gRPC에서도 Rust는 지원 안함(계획도 없어 보임)
- ReactiveX에서도 마지막 커밋은 [2015년](https://github.com/ReactiveX/RxRust).. 

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
>**자기가 참조하던 변수들과 함께 메모리에서 해제(Drop)된다.**

## 변수에 대한 Ownership/Move 개념 도입

- **Ownership은 lifetime을 결정할 수 있는 권한**
- 힙에 생성되는 변수를 다른 변수에 할당하면 Ownership은 복사되지 않고(Not Copy) 이동(Move)된다. 즉, **Owner는 언제나 1개다.**
    - Ownership을 잃어버린 변수는 uninitialized 상태가 되며, 이 변수를 다시 초기화하지 않고 사용하면 컴파일 에러
- 힙에 생성되지 않는 변수를 다른 변수에 할당하면 값 자체가 통째로 복사된다.
    - int, float, char, bool
    - int, float, char, bool를 원소로 가지는 Tuple이나 고정 크기 배열 변수

- 이건 힙에 생성되는 것이 없으므로 값이 통째로 복사되고 Ownership도 별도인 새 변수가 생긴다. 따라서 안전하고 그래서 정상 실행되지만,
    ```rust
    fn main() {
        let a: i32 = 32;
        let b = a;  // 스택 안에서 값이 통째로 복사
        println!("a is {}", a);
        println!("b is {}", b);
        
        let t1 = (23, "Jordan");
        let t2 = t1;  // 스택 안에서 값이 통째로 복사
        let t1_1_len = t1.1.len();  // t1 튜플의 두 번째(첫 번째는 0) 원소의 길이

        println!("t1_1_len() is {}", t1_1_len);
        println!("t2.1.len() is {}", t2.1.len());
    }
    ```

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

- 참고로 다른 변수에 할당할 때뿐아니라 함수에 인자로 넘길 때도, 함수에서 값을 반환할 때도 Ownership이 넘어간다. 그래서 아래와 같이 이미 Ownership을 잃은 변수 name1을 다시 사용하는 코드는 컴파일 에러가 발생한다.  
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

Rust에도 참조(Reference)가 있는데, 결국은 주소값이고 그래서 읽어오는 데는 제약이 없다는 점에서는 다른 언어의 참조와 같다. 하지만 값을 변경하는 데는 확연하게 다른 차이를 보여주는 제약 사항이 있는데, 바로 이 제약 사항이 Rust의 Thread-Safety를 보장해주는 핵심 장치다.

이에 대한 설명이 [Rust 공식 책](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)에 있지만, 읽고나서 여러모로 테스트 해 본 결과 아쉽게도 책에서 설명하지 않은 부분이 하나 있는데, 바로 **참조의 사용 여부**다. 아마도 사용되지 않는 참조가 코드에 존재하는 것 자체가 리팩터링 대상이고 결국에는 코드에서 제거되는 것이 바람직하므로, 사용 여부를 굳이 설명하지 않은 걸로 추측해본다. 하지만, **컴파일러는 참조의 실제 사용 여부를 감안해서 제약 사항의 준수 여부를 컴파일 타임에 검사**한다.

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


