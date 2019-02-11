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

## 변수에 대한 Ownership/Move 개념 도입

- Ownership은 변수의 값을 변경하고 lifetime을 결정할 수 있는 권한
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
        
        let t1 = (23, String::from("Jordan"));    // String::from("Jordan")은 힙에 생성되므로
        let t2 = t1;  //  t1의 Ownership은 t2로 이동되며, 
        let t1_1_len = t1.1.len();  // 이미 Ownership을 잃고 uninitialized 된 t1을 사용하려고 하면 컴파일 에러 발생
    }

    //-----
    error[E0382]: borrow of moved value: `t1`
      --> src/main.rs:10:20
       |
    9  |     let t2 = t1;
       |              -- value moved here
    10 |     let t1_1_len = t1.1.len();
       |                    ^^^^ value borrowed here after move
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




