# Java 개발자가 배우는 C++ - 객체의 생성

Java와 C++의 차이를 깊이 살펴보는 대신 후다닥 살펴보고, C++에 대한 이질감을 최대한 빨리 떨쳐내는 게 목적

## 객체의 생성

### 객체의 생성 방법

>Java에서는 객체의 선언만으로는 객체가 생성되지 않지만,
>
>C++에서는 객체의 선언만으로도 객체가 생성된다.

### 객체의 생성 위치

>Java에서는 Stack 영역에서 객체가 생성되지 않고 Heap 영역에서만 생성되지만,
>
>C++에서는 Stack 영역에서도 객체가 생성된다.

### 객체 생성 관련 예제

```cpp
#include <iostream>
using namespace std;

class Hello {
public:
    Hello() {
        cout << "Hello 생성자 호출" << endl;
    }

    void sayHello() {
        cout << "Hello C++" << endl;
    }
};

int main() {
    Hello hello;  // 선언만 해도 생성자가 호출된다.
    hello.sayHello();  // 선언만 하고 메소드를 호출해도 NullPointerException이 발생하지 않는다.

    return 0;
    // hello는 main 함수의 Stack에서 생성되므로
    // 별도로 hello의 메모리를 회수하지 않아도
    // main 함수의 종료와 함께 메모리에서 사라진다.
}
```

## C++의 생성자 작성 방식

### Java의 생성자

Java의 생성자는 할당 방식으로만 동작한다. 그래서 다음과 같은 코드는 컴파일 에러를 낸다.

```java
class Tmp {
  final String msg;  // 컴파일 에러 Variable 'msg' might not have been initialized

  public Tmp(String msg) {
    this.msg = msg;
  }
}
```

하지만 생성자 말고 멤버 변수 선언에서 초기화를 하면 되므로 초기활 할 방법이 없는 것은 아니다.

```java
class Tmp {
  final String msg = "This is a final msg";

  public Tmp() {
  }
}
```

### C++의 생성자

C++의 생성자도 위와 같이 작성하면 Java와 마찬가지로 할당 방식으로 동작하지만, 작성 방식에 따라 초기화 방식으로도 동작한다.

```cpp
class Tmp {
private:
  const int a;

public:
  Tmp(): a(a) {}

  int getA() {
    return a;
  }
};

int main() {
  Tmp tmp(1);

  cout << tmp.getA() << endl;
}
```

생성자를 위와 같이 기술하면 할당이 아니라 초기화 방식으로 동작하므로, 생성자에서 const 멤버 변수에 값을 초기화할 수 있다.

하지만 위 방식보다 조금 더 나은 방식은 C++11 부터 도입된 Uniform initialization 방식이다.

**쉽게 말해 다음과 같이 `()` 대신 `{}`로 값을 지정**해주는 것이다.

이왕에 개선하는 거 `const`도 개선하면 다음과 같이 바뀐다.

```cpp
#include <iostream>
using namespace std;

class Tmp {
private:
    const int a;

public:
    Tmp(const int a) : a {a} {}  // (a) 대신 {a} 사용

    // getter는 보통 뒤에 const를 붙여준다. 그래야 const object에 대해서도 호출될 수 있다.
    // 반환하는 멤버 변수가 const이므로 getter 앞에도 const 붙여줌
    const int getA() const {
        return a;
    }
};

int main() {
    Tmp tmp1 {1};  // (1) 대신 {1} 사용
    
    cout << tmp1.getA() << endl;
}
```
