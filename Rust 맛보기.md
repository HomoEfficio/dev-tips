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

Rust에도 참조(Reference)가 있는데, 결국은 주소값이라는 점에서는 다른 언어의 참조와 같지만, Ownership 관점에서는 다른 언어와 다르다.

예를 들어 자바는 다음과 같이 참조 변수를 새로운 변수에 할당하면, 주소값이 복제되면서 새로운 변수와 기존 변수 모두 Rust에서 말하는 Ownership을 갖게 된다.

```java
public static void main(String[] args) {
    final List<String> strings = new ArrayList<>();
    strings.add("a");
    strings.add("b");
    strings.add("c");
    System.out.println("strings.size(): " + strings.size());  // 3

    if (1 == 1) {
        final List<String> otherRef = strings;  // 할당되면 Ownership이 이동되는 게 아니라
        otherRef.add("d");  // otherRef과 strings 모두 ArrayList에 대한 Ownership을 가지고 수정 가능
        System.out.println("strings.size(): " + strings.size());  // 4
        final List<String> otherOtherRef = strings;
        otherOtherRef.add("e");
        System.out.println("strings.size(): " + strings.size());  // 5
        System.out.println("otherRef.size(): " + otherRef.size());  // 5
        System.out.println("otherOtherRef.size(): " + otherOtherRef.size());  // 5
    }
    // strings는 여전히 ArrayList를 가리키고 있음
    System.out.println("strings.size(): " + strings.size());  // 5
}
```

하지만 Rust의 참조(Reference)에는 Ownership이 없다. 따라서 참조는 변수의 lifetime을 결정할 수 없고 값을 빌려올 수 있을 뿐(Borrow)이다.






