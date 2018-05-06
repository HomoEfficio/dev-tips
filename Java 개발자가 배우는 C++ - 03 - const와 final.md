# Java 개발자가 배우는 C++ - const와 final

Java와 C++의 차이를 깊이 살펴보는 대신 후다닥 살펴보고, C++에 대한 이질감을 최대한 빨리 떨쳐내는 게 목적

## Java의 final

`final`을 사용하는 목적은 대략 다음과 같다.

>자료형의 앞에 붙으면 해당 자료형의 변수에 재할당 불가
>
>class 예약어 앞에 붙으면 해당 클래스 상속 불가
>
>메서드의 반환 타입 앞에 붙으면 해당 메서드 override 불가

자바의 `final`은 변수에 재할당을 허용하지 않는다. 그래서 다음과 같은 코드는 컴파일 에러가 발생한다.

```java
final int a = 3;
a = 5;  // Cannot assign a value to final variable 'a'

final int b;
b = 7;
b = 9;  // Cannot assign a value to final variable 'b'

final Object obj = new Object();
obj = new Object();  // Cannot assign a value to final variable 'obj'
```

`final`로 선언된 객체에 다른 객체를 재할당할 수는 없지만, **`final`로 선언된 객체의 상태를 바꾸는 일은 가능**하다.

```java
class Tmp {
    String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

final Tmp tmp = new Tmp();
tmp.setName("abc");  // 이렇게 객체의 메서드를 호출해서 객체의 상태를 바꾸는 것은 가능하다.
```


## C++의 const

### const object

`const T tmp`와 같이 선언되면 name 객체는 재할당이 불가하고, public 필드의 변경이나 메서드를 통한 private 필드의 변경 등 **const로 선언된 객체의 상태를 바꾸는 일도 불가능**하다. const object의 상태를 변경하지 않는 **const method**만 호출할 수 있다.

Java의 `final`과 달리 C++의 `const`는 클래스의 상속과는 관계가 없다.

```cpp
#include <iostream>
using namespace std;

class Tmp {
private:
    int a;

public:
    Tmp(int a) {
        this->a = a;
    }

    int getA() {
        return a;
    }

    void setA(int a) {
        Tmp::a = a;
    }
};

int main() {

    Tmp tmp1 {1};
    const Tmp tmp2 {2};

    tmp1 = tmp2;
    tmp2 = tmp1;  // Variable is declared constant and is not assignable
    
    tmp2.getA();  // Non-const function 'getA' is called on the const object
}
```

### const method

메서드 선언의 맨 뒤에 `const`가 붙은 메서드를 `const method`라 부른다. **`const method`는 객체의 상태를 바꾸지 않음을 보장**한다.

따라서 getter 메서드는 `const method`로 만들어서 외부에서 호출할 수 있지만, 객체의 상태를 변경하는 setter 메서드에 `const`를 붙이면 다음과 같이 컴파일 에러가 난다.

```cpp
#include <iostream>
using namespace std;

class Tmp {
private:
    int a;

public:
    Tmp(int a) {
        this->a = a;
    }

    int getA() const {
        return a;
    }

    void setA(int a) const {
        this->a = a;  // Field 'a' is assigned inside a const function
    }
};

int main() {

    Tmp tmp1 {1};
    const Tmp tmp2 {2};

    tmp1 = tmp2;
    tmp2 = tmp1;  // Variable is declared constant and is not assignable
    
    tmp2.getA();  // OK
}
```

자바의 `final`과 달리 C++의 **`const`는 메서드의 override와 관계가 없고, 값이나 상태의 변경 여부와 관계 있다.**


### 메서드 선언 맨 앞에 붙는 const

반환되는 데이터를 `const`로 만든다. `const`로 반환된 값은 `non-const`에도 할당될 수 있고, 이렇게 되면 불변성을 상실한다.

```cpp
#include <iostream>
using namespace std;

class Embedded {
private:
    long b;

public:
    Embedded(long b) {
        this->b = b;
    }

    long getB() const {
        return b;
    }

    void setB(long b) {
        Embedded::b = b;
    }
};

class Tmp {
private:
    Embedded emb;

public:
    Tmp(const Embedded &emb) : emb(emb) {}

    const Embedded getEmb() const {
        return this->emb;
    }

    void setA(Embedded emb) {
        this->emb = emb;
    }
};

int main() {

    Embedded emb {99L};
    Tmp tmp {emb};

    Embedded nonConst = tmp.getEmb();  // tmp.getEmb()가 const Embedded를 반환하더라도
    nonConst.setB(100L);  // const가 아닌 변수에 할당되면

    cout << nonConst.getB() << endl;  // const가 아닌 변수에 의해 상태 변경 가능
}
```

### 포인터에 사용되는 const

C++의 **`const`은 포인터의 자료형의 앞 뒤 모두에 올 수 있으며, `const` 바로 뒤에 있는 요소에 재할당을 금지**한다.

```cpp
int main() {

    int c = 2;

    // 포인터인 pC1에는 다른 값을 재할당 가능, 포인터가 가리키는 곳인 *pC1에는 다른 값 재할당 불가
    const int* pC1 = &c;

    // 포인터인 pC1에는 다른 값을 재할당 불가능, 포인터가 가리키는 곳인 *pC1에는 다른 값 재할당 가능
    int* const pC2 = &c;
    
    int d = 4;

    pC1 = &d;
    // const int* 이므로 *를 붙여 포인터가 가리키는 곳에 재할당 불가
    *pC1 = 6;  // 'const int *' is read-only pointer
    
    // int* const pC2 이므로 pC2에 재할당 불가
    pC2 = &d;  // variable is declared constant and is not assignable
    *pC2 = 6;
}
```

## C++ const 정리

>상속이나 override와도 관련이 있는 자바의 `final`과는 달리 C++의 `const`는 데이터 불변화 관련 역할만 담당
>
>const object, const method, 반환타입 앞 const, const pointer
>
>C++11부터 자바의 `final`과 유사한 역할을 하는 `sealed`와 `final` 제공
