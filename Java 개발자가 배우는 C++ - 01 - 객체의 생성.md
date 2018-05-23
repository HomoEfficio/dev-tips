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

Java에서는 생성자 안에서도 멤버 변수를 초기화 할 수 있다.

```java
class Tmp {
  private final String msg;

  public Tmp(String msg) {
    // 이미 생성되어 있는 this.msg 라는 변수에 msg를 할당하는 것처럼 보이지만,
    // 할당할 수 없는 final 변수인 this.msg에 할당이 성공되는 것으로 봐서
    // 할당이 아니라 초기화로 동작함을 짐작할 수 있다.
    this.msg = msg;  
  }
}
```


### C++의 생성자

Java에서는 생성자를 작성하는 방식이 한 가지지만, C++의 생성자 작성 방식은 여러가지다.

#### 할당 방식

C++의 생성자를 Java의 생성자 작성 방식처럼 작성하면 생성자의 body 내부 코드는 할당 방식으로 동작한다. 따라서 다음과 같은 코드는 컴파일 되지 않는다.

```cpp
#include <iostream>
using namespace std;

class OtherTmp {
private:
  const int b;

public:
  OtherTmp(const int b) : b {b} {}

  const int getB() const {
    return b;
  }
};

class Tmp {
private:
  const int a;
  OtherTmp& otherTmp;

public:
  Tmp(const int a, OtherTmp& otherTmp) {
    this->a = a;  // const 변수인 this->a 에 할당을 시도하므로 컴파일 에러 발생
    this->otherTmp = otherTmp;  // 할당이 불가능한 Reference에 할당을 시도하면 default operator=에 의해 의도하지 않은 Shallow Copy 발생
  }
  
  int getA() {
    return a;
  }
  
  OtherTmp &getOtherTmp() const {
    return otherTmp;
  }
};

int main() {
  OtherTmp otherTmp {3};
  OtherTmp& otherTmpRef = otherTmp;
  Tmp tmp1 {1, otherTmpRef};

  cout << tmp1.getA() << endl;
  cout << tmp1.getOtherTmp().getB() << endl;
}

```

위와 같이 할당 방식으로 값을 지정해 줄 수 없을 때는 constructor-initializer(ctor-initializer)를 사용해야 한다.

```cpp
//
// Created by homoefficio on 18. 5. 6.
//
#include <iostream>
using namespace std;

class OtherTmp {
private:
  const int b;

public:
  OtherTmp(const int b) : b {b} {}

  const int getB() const {
    return b;
  }
};

class Tmp {
private:
  const int a;
  OtherTmp& otherTmp;

public:
  Tmp(const int a, OtherTmp& otherTmp) : a {a}, otherTmp {otherTmp} {}  // ctor-initializer

  const int getA() const {
    return a;
  }

  OtherTmp &getOtherTmp() const {
    return otherTmp;
  }
};

int main() {
  OtherTmp otherTmp {3};
  OtherTmp& otherTmpRef = otherTmp;
  Tmp tmp1 {1, otherTmpRef};

  cout << tmp1.getA() << endl;
  cout << tmp1.getOtherTmp().getB() << endl;
}
```

생성자를 위와 같이 C++11 부터 도입된 **Uniform initialization**(**쉽게 말해 다음과 같이 `()` 대신 `{}`로 값을 지정**)을 활용해서 기술하면 할당이 아니라 초기화 방식으로 동작하므로, 생성자에서 const 멤버 변수에 값을 초기화할 수 있다.


## Default 생성자

### 컴파일러가 만들어 주는 Default 생성자

C++에서도 Java와 마찬가지로 개발자가 어떤 클래스에 아무런 생성자도 정의하지 않으면, 컴파일러가 자동으로 Default 생성자를 만들어 준다.

Default 생성자는 `MyClass myClass;`와 같이 아무 인자 없이 객체 선언만으로 Stack 영역에 객체를 만들 때와 객체 배열을 만들 때 등 아무 인자 없이 객체를 생성할 때 호출된다.

### 개발자가 명시해 주는 Default 생성자

클래스에 개발자가 작성한 생성자가 있으면, 컴파일러는 Default 생성자를 자동으로 만들어주지 않으므로 개발자가 작성해야 한다. 이 때 클래스 헤더 파일에서 다음과 같이 `default` 예약어를 사용하면 클래스 구현 파일에 별도로 Default 생성자를 작성하지 않아도 Default 생성자가 자동으로 만들어진다.

```cpp
#include <iostream>
using namespace std;

class Tmp {
private:
    const int a;

public:
    Tmp() = default;  // default 예약어로 Default 생성자 명시
    Tmp(const int a) : a{a} {}

    const int getA() const {
        return a;
    }
};

int main() {
    Tmp tmp1 {1};  // (1) 대신 {1} 사용
    
    cout << tmp1.getA() << endl;
}
```

## 복사 생성자

- 언제 호출 되나
  - 객체가 인자로 전달될 때
  - 객체가 반환값으로 반환될 때

## operator=

- default


