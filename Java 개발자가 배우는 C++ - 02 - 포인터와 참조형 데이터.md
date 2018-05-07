# Java 개발자가 배우는 C++ - 포인터와 참조형 데이터

Java와 C++의 차이를 깊이 살펴보는 대신 후다닥 살펴보고, C++에 대한 이질감을 최대한 빨리 떨쳐내는 게 목적

## 포인터와 참조형 데이터

Java는 기본형(primitive) 데이터 외에는 모두 참조형(reference) 데이터다.

C++에는 기본형, 포인터, 참조형 데이터가 있다.

## 포인터

C++의 포인터는 어떤 데이터의 메모리 주소값을 담는 변수다.

주소 연산자인 `&`를 변수 앞에 붙여주면, 그 변수의 메모리 주소값을 읽어올 수 있다.

포인터 변수에 담긴 주소에 저장된 값을 읽어오려면 포인터 변수 앞에 `*`를 붙여주면 된다.

```cpp
int a = 1;
int* p = &a;  // a의 주소를 포인터 변수 p에 할당
cout << *p << endl;  // p에 담긴 주소에 저장된 값인 1을 출력
```

포인터에 담긴 주소값은 연산에 사용할 수도 있다.

```cpp
// 대략 아래와 같이 포인터에 담긴 주소값을 연산에 사용할 수 있다.
char* pointer = const_cast<char *>("ABCDE");  // "ABCDE"가 저장된 메모리 주소를 pointer 변수에 담는다.
cout << *pointer << endl;  // A 출력
cout << *(pointer + 2 * sizeof(char)) << endl;  // C 출력
```

## Java의 참조형 변수와 C++의 포인터

>Java의 참조형 변수도 내부적으로 주소값을 담는다는 점에서 C++의 포인터와 비슷하지만, Java에서는 참조형 변수에 담긴 주소값으로 연산을 할 수 없다.
>
>C++의 포인터는 `int* a`에서 int를 가리키는 a를 `char*`로 캐스팅 할 수도 있지만, Java의 참조형 변수는 원래 가리키던 타입만 가리킬 수 있으므로 Java의 참조형 변수는 strong type이다.


## C++의 포인터와 참조형 변수

포인터와 참조형 변수는 같은 점도 있고 다른 점도 있다.

둘 모두 주소값을 다룰 수 있다는 점에서는 같지만 그 외에는 많이 다르다.

일단 아래와 같이 포인터와 참조형 변수를 사용하는 코드로 비교해보면 표면적으로 쉽게 알 수 있다.

```cpp
int a = 1;

int* pA;  // 포인터는 초기화 없이 선언만 할 수 있다.
pA = &a;  // 선언한 후에 값을 할당할 수 있다. 
// 포인터에는 주소 연산자 &가 붙은 주소값을 할당한다.
cout << *pA << endl;  // 포인터가 가리키는 주소에 저장된 값을 출력하려면 * 연산자를 포인터 변수 앞에 붙여줘야 한다.

int& rA = a;  // 참조형 변수는 선언과 함께 반드시 초기화해줘야 한다. 초기화 안 해주면 컴파일 에러.
// 참조형 변수에는 주소 연산자 &가 붙어 있지 않은 그냥 변수명을 할당한다.
cout << rA << endl;  // 참조형 변수는 별칭이므로 그냥 변수명 그대로 출력하면 참조형 변수가 가리키는 값이 출력된다.
```

>참조형 변수는 한 마디로 어떤 변수에 대한 별칭(alias)이다.

조금 속 깊은 관점에서 포인터와 참조형 변수의 차이는 다음과 같다.

>포인터의 포인터는 존재할 수 있지만,
>
>레퍼런스의 레퍼런스, 레퍼런스의 배열, 레퍼런스의 포인터는 존재할 수 없다.

한 꺼풀 더 들어가 보면, 참조형 변수에 값을 초기화하는 것이 아니라 선언 후 값을 할당할 때 벌어지는 일도 살펴볼만하다. 다만, **실제 실무에서 참조형 변수는 항상 초기화해서 사용한다**는 규칙을 철저히 지킨다면 다음의 장황한 내용은 사실 그냥 넘어가도 된다.

>포인터는 동일한 타입의 다른 데이터를 가리키도록 다른 값을 할당 받을 수 있지만,
>
>참조형 변수는 동일한 타입의 다른 데이터를 가리키도록 다른 값을 할당 받을 수 없다.
>
>- 참조형 변수에 동일한 타입의 다른 데이터를 할당하면,
>- 참조형 변수가 다른 데이터를 가리키는 게 아니라,
>- 다른 데이터의 얕은 복사본이 참조형 변수가 가리키던 데이터를 덮어쓴다.

복잡해 보이므로 코드를 통해 알아보자.

## 포인터와 참조형 변수 통합 예제

```cpp
#include <iostream>
#include <string>
using namespace std;

class Hello {
public:
    Hello() {
        cout << "Hello 디폴트 생성자 호출" << endl;
        greeting = "Hello";
    }

    Hello(string s) {
        cout << "Hello 생성자 호출" << endl;
        greeting = s;
    }

    void sayHello(string s) {
        cout << greeting << " " << s << endl;
    }
private:
    string greeting;
};

int main() {
    Hello hello;  // 선언만 해도 생성자가 호출된다.
    hello.sayHello("C++");  // 선언만 하고 메소드를 호출해도 NullPointerException이 발생하지 않는다.
    cout << "hello 객체의 주소: " << &hello << "    (1)" << endl;  // (1)

    Hello* pHello = &hello;
    pHello->sayHello("Pointer *****");
    cout << "pHello가 가리키는 객체의 주소: " << pHello << "    (2)" << endl;  // (2)
    cout << "pHello의 주소: " << &pHello << "    (3)" << endl;  // (3)

    Hello& rHello = hello;
    rHello.sayHello("Reference &&&&&");
    cout << "rHello가 가리키는 객체의 주소: " << &rHello << "    (4)" << endl;  // (4)
    // rHello의 주소는 표현할 방법이 없다.

    cout << endl;

    Hello hi {"Hi"};
    cout << "hi 객체의 주소: " << &hi << "    (5)" << endl;  // (5)

    pHello = &hi;
    pHello->sayHello("pHello에 hi 객체 주소 할당 후 *****");
    cout << "pHello가 가리키는 객체의 주소: " << pHello << "    (6)" << endl;  // (2)과 다른 값이 출력된다.
    cout << "pHello의 주소: " << &pHello << "    (7)" << endl;  // (3)과 같은 값이 출력된다.

    rHello = hi;
    rHello.sayHello("rHello에 hi 할당 후 &&&&&");
    cout << "rHello가 가리키는 객체의 주소: " << &rHello << "    (8)" << endl;  // (4)와 같은 값이 출력된다.

    cout << endl;

    cout << "hello로 초기화 되어있던 rHello에 hi을 할당한 후, hello.sayHello(\"???\")를 실행하면 뭐가 출력될까?" << endl;
    cout << "Hello ??? 일까, Hi ??? 일까" << endl;
    cout << "hello.sayHello(\"???\"): ";
    hello.sayHello("???");  // Hello로 출력될까 Hi로 출력될까?
    cout << "hello 객체의 주소: " << &hello << "    (9)" << endl;  // (1)과 같은 값이 출력된다.
    cout << "hello 객체의 주소는 예전과 동일한데 내용이 바뀌었다!!" << endl;
    cout << "hello 객체를 가리키던 reference인 rHello에 hi 객체를 할당하면서" << endl;
    cout << "rHello가 hi 객체를 가리키도록 재할당되는 것이 아니라" << endl;
    cout << "hello 객체가 있던 자리에 hi 객체의 얕은 복사본이 들어가버림" << endl;

    return 0;
}
```

실행 결과는 다음과 같다. **참조형 변수에 재할당 시 얕은 복사가 발생하고 원래 값을 덮어씀**을 알 수 있다.

```
Hello 디폴트 생성자 호출
Hello C++
hello 객체의 주소: 0x7ffe90146590    (1)
Hello Pointer *****
pHello가 가리키는 객체의 주소: 0x7ffe90146590    (2)
pHello의 주소: 0x7ffe90146570    (3)
Hello Reference &&&&&
rHello가 가리키는 객체의 주소: 0x7ffe90146590    (4)

Hello 생성자 호출
hi 객체의 주소: 0x7ffe90146610    (5)
Hi pHello에 hi 객체 주소 할당 후 *****
pHello가 가리키는 객체의 주소: 0x7ffe90146610    (6)
pHello의 주소: 0x7ffe90146570    (7)
Hi rHello에 hi 할당 후 &&&&&
rHello가 가리키는 객체의 주소: 0x7ffe90146590    (8)

hello로 초기화 되어있던 rHello에 hi을 할당한 후, hello.sayHello("???")를 실행하면 뭐가 출력될까?
Hello ??? 일까, Hi ??? 일까
hello.sayHello("???"): Hi ???
hello 객체의 주소: 0x7ffe90146590    (9)
hello 객체의 주소는 예전과 동일한데 내용이 바뀌었다!!
hello 객체를 가리키던 reference인 rHello에 hi 객체를 할당하면서
rHello가 hi 객체를 가리키도록 재할당되는 것이 아니라
hello 객체가 있던 자리에 hi 객체의 얕은 복사본이 들어가버림

Process finished with exit code 0
```

참조형 변수에 다른 값을 할당할 때 할당 대신 얕은 복사가 일어나고 원래 값이 덮어써지는 현상은 상당히 낯설지만, **실제 실무에서는 참조형 변수에 다른 값을 할당하지 않으면 되므로 현실적으로는 크게 신경쓸 필요가 없다.**

**다른 값을 할당해야 한다면 참조형 변수를 쓰지 말고 포인터를 쓰면 된다.**

C++에 포인터가 있긴 하지만, **C++에서도 가능하다면 포인터보다는 참조형 변수 사용을 권장하며, 포인터는 어쩔 수 없이 꼭 써야할 떄만 쓰라고 권장 한다.**

>Use references when you can, and use pointers when you have to.

참고: https://www.geeksforgeeks.org/pointers-vs-references-cpp/



## Java의 참조형 변수와 C++의 포인터 및 참조형 변수

>Java의 참조형 변수는 다른 주소값을 할당 받을 수 있다는 점에서 C++의 포인터와 비슷하고,
>
>Java의 참조형 변수에 담긴 주소값을 명시적으로 직접 접근하거나 연산할 수 없다는 점에서 C++의 참조형 변수와 비슷하다.


## 스마트 포인터

C++에는 세 가지 스마트 포인터가 있다.

1. `std::unique_ptr`
2. `std::shared_ptr`
3. `std::weak_ptr`

참고로 `auto_ptr`은 deprecated 되었다.

### unique_ptr

이 중 가장 많이 사용되는 **`unique_ptr`은 Heap에 할당된 동적 메모리를 delete로 해제해주지 않아도 해당 포인터 변수의 스코프를 벗어나면 자동으로 메모리를 해제**해준다.

C++14에서는 아래와 같이 `make_unique` 함수를 통해 `unique_ptr` 변수를 생성할 수 있고,

`auto anEmployee = std::make_unique<Employee>();`

C++14 미만에서는 아래와 같이 `unique_ptr`로 만들 수 있다.

`std::unique_ptr<Employee> anEmployee(new Employee);`

### shared_ptr

나중에

### weak_ptr

나중에



