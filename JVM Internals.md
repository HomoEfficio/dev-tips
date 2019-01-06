# JVM Internals

> 원문: https://blog.jamesdbloom.com/JVMInternals.html

Java Virtual Machine(JVM)의 내부 아키텍처를 알아보자. 아래의 다이어그램은 Java 7 SE를 준수하는 전형적인 JVM의 주요 내부 구성요소를 보여주고 있다.

![Imgur](https://i.imgur.com/DeLfwDA.png)

그림에 나오는 구성요소는 두 개의 단원으로 나누어 설명한다. 첫 번째 단원에서는 각 스레드별로 생성되는 구성요소를 다루고, 두 번째 단원에서는 스레드와 관계 없이 독립적으로 생성되는 구성요소를 다룬다.

# 스레드

스레드는 프로그램의 실행 단위다. JVM은 애플리케이션이 여러 개의 스레드를 동시에 실행할 수 있도록 허용한다. HotSpot JVM에서는 자바 스레드와 네이티브 OS 스레드 사이에 직접적인 매핑이 존재한다. 자바 스레드의 상태를 구성하는 ThreadLocal 저장소, 버퍼 할당, 객체 동기화, 스택, 프로그램 카운터(PC)가 모두 준비된 후에 네이티브 스레드가 생성된다. 네이티브 스레드는 자바 스레드가 종료되면 회수된다. OS는 모든 스레드를 스케줄링하고 가용한 CPU로 스레드를 디스패치하는 일을 담당한다. 네이티브 스레드가 초기화되면 자바 스레드의 `run()` 메서드를 호출한다. `run()` 메서드가 반환되거나 잡히지 않은 예외가 처리되고 나면, 네이티브 스레드는 스레드의 종료와 함께 JVM도 종료되어야 하는지(즉, 스레드가 마지막 non-daemon 스레드인지) 확인한다. 스레드가 종료되면 네이티브 스레드와 자바 스레드에 사용된 모든 리소스는 반납된다.

## JVM 시스템 스레드

jconsole이나 어떤 디버거를 사용하면 백그라운드로 많은 스레드가 실행되는 것을 볼 수 있다. 이런 백그라운드 스레드는 `public static void main(String[])`를 호출하는 과정에서 생성되는 메인 스레드와 메인 스레드에 의해 생성되는 스레드에 덧붙여 함께 실행된다. HotSpot JVM에 있는 주요 백그라운드 시스템 스레드는 다음과 같다.

스레드 이름 | 설명
--- | ---
VM thread | JVM이 safe-point에 도달하게 만드는 연산을 기다리는 스레드. JVM이 safe-point에 도달하게 만드는 연산이 분리된 스레드에 발생해야 하는 이유는 그런 연산 모두가 JVM이 힙(heap)에 대한 수정이 발생할 수 없는 safe-point 상태가 되도록 This thread waits for operations to appear that require the JVM to reach a safe-point. The reason these operations have to happen on a separate thread is because they all require the JVM to be at a safe point where modifications to the heap can not occur. The type of operations performed by this thread are "stop-the-world" garbage collections, thread stack dumps, thread suspension and biased locking revocation.
Periodic task thread | 주기적으로 실행되는 연산을 스케줄링하는 데 사용되는 타이머 이벤트(즉, 인터럽트)를 담당하는 스레드.
GC threads | JVM에서 발생하는 유형별 가비지 컬렉션을 담당하는 스레드.
Compiler threads | 런타임에 바이트 코드를 네이티브 코드로 컴파일 하는 스레드.
Signal dispatcher thread | JVM 프로세스에 전달된 시그널을 받고 적절한 JVM 메서드를 호출해서 JVM 내에서 시그널을 처리하는 스레드.

## Per Thread

실행되는 스레드는 각각 다음의 구성요소를 가지고 있다.

### 프로그램 카운터(Program Counter, PC)

JVM은 멀티 스레드의 동시 실행을 지원한다. JVM 스레드는 PC 레지스터를 가지고 있다. 어느 특정 순간으로 잘라보면, JVM 스레드는 하나의 메서드에 포함된 코드를 실행하고 있는데, 이 메서드를 그 스레드의 현재 실행 중인 메서드(current method)라고 부른다. 현재 실행 중인 메서드가 네이티브 메서드가 아니라면 PC에는 현재 실행 중인 JVM 명령어(JVM instruction)의 주소가 저장된다. 현재 실행 중인 메서드가 네이티브 메서드이면 PC의 값은 정해지지 않는다. 모든 CPU는 PC를 가지고 있고, PC에 저장된 값은 일반적으로 각 명령어가 실행된 후에 증가하므로, 실행 후에는 다음에 실행될 명령어의 주소가 PC에 저장된다. JVM은 PC를 이용해서 명령어가 실행되는 위치를 추적한다. PC는 실제로 메서드 영역(Method Area)에 있는 메모리 주소를 가리킨다.

### 스택(Stack)

각 스레드는 자기만의 스택을 가지며, 스택에는 해당 스레드에서 실행되는 메서드의 프레임(frame)이 저장된다. 프레임은 메서드가 새로 호출될 때마다 새로 생성되고, 스택은 후입선출(LIFO, Last In First Out)의 자료구조이므로 현재 실행되고 있는 메서드의 프레임이 스택의 맨 위에 위치한다. 프레임은 해당 메서드가 정상적으로 값을 반환하거나 메서드 실행 과정에서 잡히지 않은 예외가 던져지면 스택에서 꺼내어진다. 스택은 프레임 객체를 넣고 빼는 일을 제외하면 직접적으로 조작되지 않으며, 따라서 프레임 객체는 힙에 할당될 수도 있고 메모리는 연속적으로 인접해 있지 않아도 된다.

### 네이티브 스택(Native Stack)

모든 JVM이 네이티브 메서드를 지원하는 것은 아니다. 하지만 네이티브 메서드를 지원하는 JVM은 스레드 별로 네이티브 메서드 스택을 생성한다. JVM이 JNI(Java Native Invocation)를 위해 C 언어의 링크 모델을 사용해서 구현되었다면, 네이티브 스택은 C 스택이다. 이런 경우 인자와 반환값의 순서는 일반적인 C 프로그램의 네이티브 스택에서와 동일하다. 네이티브 메서드는 (JVM 구현에 따라 다를 수 있지만) 일반적으로 JVM 내부의 자바 메서드를 호출할 수 있다. 이런 네이티브로부터 자바로 향하는 호출은 보통의 자바 스택 상에서 발생하며, 스레드는 네이티브 스택을 떠나서 보통의 자바 스택에 프레임을 생성해서 맨 위에 추가한다.

### 스택 제약사항(Stack Restrictions)

스택 크기는 동적으로 변할 수도 있고 고정 크기일 수도 있다. 스레드가 허용된 크기보다 더 큰 스택을 필요로 하면 StackOverflowError 가 던져진다. 스레드가 새 프레임을 필요로 하는데 남아 있는 메모리가 충분하지 않다면 OutOfMemoryError 가 던져진다.

### 프레임

새 프레임은 메서드 호출마다 새로 생성되어 스택의 맨 위에 추가되고, 메서드가 정상적으로 값을 반환하거나 메서드 실행 과정에서 잡히지 않은 예외가 던져지면 스택에서 꺼내어진다. 예외 처리는 아래의 [예외 테이블(Exception Tables)]()을 참고한다.

각 프레임은 다음의 구성요소를 가지고 있다.

- 로컬 변수 배열(Local variable array)
- 반환 값(Return value)
- 오퍼랜드 스택(Operand stack)
- 현재 실행되는 메서드가 속한 클래스의 런타임 상수 풀에 대한 참조(Reference to runtime constant pool for class of the current method)

### 로컬 변수 배열

로컬 변수 배열에는 메서드의 실행에 사용되는 모든 변수가 저장된다. `this`에 대한 참조, 메서드의 파라미터, 로컬에서 정의된 변수는 모두 로컬 변수 배열에 저장된다. 클래스 메서드(정적 메서드)의 파라미터는 배열 인덱스 0부터 저장되며, 인스턴스 메서드인 경우 배열 인덱스 0에는 `this`에 대한 참조가 저장된다.

로컬 변수는 다음과 같다.

- `boolean`
- `byte`
- `char`
- `long`
- `short`
- `int`
- `float`
- `double`
- `reference`
- `returnAddress`

`long`과 `double`을 제외한 모든 타입은 로컬 변수 배열에서 한 개의 슬롯(slot)을 차지한다. `long`과 `double`은 두 개의 슬롯을 차지한다.

### 오퍼랜드 스택

오퍼랜드 스택은 바이트코드 명령어의 실행 중에 사용되는데, 네이티브 CPU에서 범용 레지스터가 사용되는 방식과 유사하다. 대부분의 JVM 바이트코드는 오퍼랜드 스택에 넣고(push), 꺼내고(pop), 복사하고(duplicate), 스왑(swap)하거나 값을 생산하거나 소비하는 연산을 실행하는데 대부분의 시간을 사용한다. 그래서 로컬 변수 배열과 오퍼랜드 스택 사이에서 값을 이동하는 명령어는 바이트코드에 자주 등장한다. 예를 들어, 다음과 같은 단순한 변수 초기화는

```
int i;
```

다음과 같이 오퍼랜드 스택과 상호작용하는 두 개의 바이트코드로 컴파일된다.

```
0:    iconst_0    // 0을 오퍼랜드 스택의 맨 위에 추가
1:    istore_1    // 오퍼랜드 스택의 맨 위에서 값을 꺼내서 로컬 변수 1에 저장한다.
```

로컬 변수 배열과 오퍼랜드 스택, 런타임 상수 풀 사이의 상호작용에 대한 자세한 내용은 [클래스 파일 구조]()를 참고한다.

### 동적 링크

프레임은 런타임 상수 풀에 대한 참조를 가지고 있다. 이 참조는 해당 프레임에서 실행되는 메서드가 속한 클래스의 위한 상수 풀을 가리킨다.

일반적으로 C/C++ 코드는 object 파일로 컴파일 되고, 다수의 object 파일은 함께 링크되어 하나의 실행 파일이나 dll 같은 사용 가능한 형태의 결과물을 만들어낸다. 링크 단계에서 각 object 파일 안에 있는 심볼릭 참조(symbolic references)는 실행 가능한 최종 결과물의 실제 메모리 상에서의 상대 주소로 대체된다. 자바에서 이 링크 단계는 런타임에 동적으로 수행된다.

자바 클래스가 컴파일 될 때 변수와 메서드에 대한 모든 참조는 클래스의 상수 풀에 심볼릭 참조로 저장된다. 심볼릭 참조는 논리적 참조이며 실제 물리적인 메모리 위치를 참조하지 않는다. JVM 구현체는 심볼릭 참조를 언제 해석(resolve)할지 선택할 수 있다. 심볼릭 참조의 해석은 클래스가 로딩된 후 확인 과정에서 수행될 수 있으며 이런 방식의 해석을 즉시(eager) 또는 정적(static) 해석이라고 부른다. 이와는 달리 심볼릭 참조가 처음으로 실제 사용될 때 해석될 수도 있는데 이런 해석을 지연(lazy) 또는 늦은(late) 해석이라고 부른다. 하지만 JVM은 각 참조가 처음으로 사용될 때 해석되는 것처럼 동작해야 하고, 이 지점에서 발생하는 해석 오류를 던져야 한다. 바인딩(binding)은 심볼릭 참조로 식별되는 필드, 메서드, 클래스가 직접 참조로 대체되는 과정을 말한다. 심볼릭 참조가 아직 해석되지 않은 클래스를 가리킨다면, 이 클래스는 로딩된다. 직접 참조는 런타임에서의 변수나 메서드의 위치와 연관되는 저장 구조에 대한 오프셋(offset)의 형태로 저장된다.


## 스레드 사이에서 공유되는 것

### 힙(Heap)

힙은 런타임에서 클래스의 인스턴스나 배열을 할당하는 데 사용된다. 배열과 객체(object)는 절대로 스택에 저장되지 않는데, 이는 프레임은 생성된 후에 크기를 변경할 수 없도록 설계되었기 때문이다. 프레임은 힙에 존재하는 객체나 배열을 가리키는 참조만을 저장한다. 기본형 변수나 각 프레임 안에 존재하는 로컬 변수 배열에 있는 참조와는 달리, 객체는 항상 힙에 저장되므로 메서드가 종료될 때 함께 제거되지 않고, 가비지 컬렉터에 의해 제거된다.

가비지 컬렉션을 지원하기 위해 힙은 세 가지 generation으로 구분된다.

- Young Generation
    - Young Generation은 다시 Eden과 Survivor로 분류될 수 있다.
- Old Generation(Tenured Generation)
- Permanent Generation

### 메모리 관리

객체와 배열은 명시적인 방법으로는 절대 제거되지 않으며 가비지 컬렉터에 의해 자동으로 회수된다.

메모리 관리는 일반적으로 다음의 과정에 따라 수행된다.

1. 새 객체와 배열은 Young Generation 안에 생성된다.
1. Young Generation 안에서는 마이너(minor) 가비지 컬렉션이 수행된다. 마이너 가비지 컬렉션 후에도 제거되지 않는 객체는 Eden 공간에서 Survivor 공간으로 옮겨진다.
1. 메이저(major) 가비지 컬렉션은 일반적으로 스레드의 중단을 유발하며, 객체가 generation 수준에서 이동된다. 메이저 가비지 컬렉션 후에도 제거되지 않은 객체는 Young Generation에서 Old(Tenured) Generation으로 옮겨진다.
1. Permanent Generation의 가비지 컬렉션은 Old Generation에서 가비지 컬렉션이 수행될 때 함께 수행된다. 둘 중의 한 쪽이 가득차면 둘 모두에서 가비지 컬렉션이 수행된다.

### Non-Heap 메모리

JVM 동작의 일부로 간주되는 객체는 힙에 생성되지 않는다.

Non-Heap 메모리는 다음을 포함한다.

- 메서드 영역(Method Area)과 Interned String을 포함하는 Permanent Generation
- 코드 캐시(Code Cache)는 JIT 컴파일러에 의해 네이티브 코드로 컴파일 된 메서드의 컴파일과 저장에 사용된다.

### Just In Time(JIT) 컴파일

자바 바이트코드는 인터프리트(interprete) 되지만 JVM의 호스트 CPU에서 직접 실행되는 네이티브 코드만큼 빠르지 않다. 이런 부족한 성능을 개선하기 위해 오라클 HotSpot VM은 규칙적으로 실행되는 바이트코드 영역인 hot 영역을 찾아서 네이티브 코드로 컴파일한다. 이 네이티브 코드는 Non-Heap 영역에 있는 코드 캐시에 저장된다. 이를 통해 HotSpot VM은 컴파일에 소요되는 추가적인 시간과 코드를 인터프리트하는 데 소요되는 시간의 손익(trade-off)을 잘 따져서 가장 적절한 방법을 선택하기 위해 노력한다.

### 메서드 영역(Method Area)

메서드 영역은 클래스 단위로 저장되는 정보를 저장한다.

- 클래스로더에 대한 참조
- 런타임 상수 풀
    - 숫자 상수
    - 필드 참조
    - 메서드 참조
    - 애트리뷰트(자세한 내용은 [JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.7) 참고)
- 필드 데이터
    - 이름
    - 타입
    - 수정자(Modifiers)
    - 애트리뷰트(자세한 내용은 [JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.7) 참고)
- 메서드 데이터
    - 이름
    - 반환 타입
    - 파라미터 타입
    - 수정자(Modifiers)
    - 애트리뷰트(자세한 내용은 [JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.7) 참고)
- 메서드 코드
    - 바이트코드
    - 오퍼랜드 스택 크기
    - 로컬 변수 크기
    - 로컬 변수 테이블
    - 예외 테이블
        - 시작 지점
        - 종료 지점
        - 예외 핸들러 코드에 대한 PC 오프셋
        - 잡힌 예외 클래스에 대한 상수 풀 인덱스

모든 스레드는 동일한 메서드 영역을 공유하므로, 메서드 영역에 저장된 데이터에 대한 접근과 동적 링크 과정은 반드시 스레드 세이프(Thread-safe)해야 한다. 만약 두 개의 스레드가 아직 로딩되지 않은 클래스의 필드나 메서드에 접근을 시도하면, 클래스는 오직 한 번만 로딩되어야 하며 로딩이 완료될 때까지 두 스레드의 실행은 지속되지 않아야 한다.

### 클래스 파일 구조

컴파일된 클래스 파일은 다음과 같은 구조로 구성된다. (역주: 원문 설명이 부족하면 [JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.1)을 참고해도 좋다)

```
ClassFile {
    u4            magic;
    u2            minor_version;
    u2            major_version;
    u2            constant_pool_count;
    cp_info        contant_pool[constant_pool_count – 1];
    u2            access_flags;
    u2            this_class;
    u2            super_class;
    u2            interfaces_count;
    u2            interfaces[interfaces_count];
    u2            fields_count;
    field_info        fields[fields_count];
    u2            methods_count;
    method_info        methods[methods_count];
    u2            attributes_count;
    attribute_info    attributes[attributes_count];
}
```

항목 | 설명
--- | ---
magic, minor_version, major_version | 클래스의 버전 정보와 이 클래스를 컴파일 한 JDK의 버전
constant_pool | 심볼 테이블과 비슷하지만 더 많은 데이터를 담는다. 자세한 내용은 [런타임 상수 풀]()에서 다룬다.
access_flags | 클래스의 수정자(modifiers) 표시
this_class | constant_pool 에서 클래스의 전체 이름(org/jamesdbloom/foo/Bar)을 가리키는 인덱스 값
super_class | constant_pool 에서 수퍼 클래스(java/lang/Object)에 대한 심볼릭 참조를 가리키는 인덱스 값
interfaces | constant_pool 에서 클래스가 구현하는 모든 인터페이스에 대한 심볼릭 참조를 가리키는 인덱스 값의 배열
fields | constant_pool 에서 클래스의 각 필드에 대한 완전한 설명을 가리키는 인덱스 값의 배열
methods | constant_pool 에서 클래스의 각 메서드에 대한 완전한 설명을 가리키는 인덱스 값의 배열. 추상 메서드나 네이티브 메서드가 아니면 바이트코드도 완전한 설명에 포함된다.
attributes | 클래스에 대한 추가적인 정보를 제공하는 값들의 배열. RetentionPolicy.CLASS나 RetentionPolicy.RUNTIME인 애노테이션도 추가적인 정보에 포함된다.

`javap` 명령을 사용하면 컴파일된 자바 클래스 파일을 바이트코드를 볼 수 있다.

다음과 같은 단순한 클래스를 컴파일하고,

```java
package org.jvminternals;

public class SimpleClass {

    public void sayHello() {
        System.out.println("Hello");
    }

}
```

다음 명령을 실행하면,

>javap -v -p -s -sysinfo -constants classes/org/jvminternals/SimpleClass.class

다음과 같은 바이트코드를 볼 수 있다.

```java
public class org.jvminternals.SimpleClass
  SourceFile: "SimpleClass.java"
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#17         //  java/lang/Object."<init>":()V
   #2 = Fieldref           #18.#19        //  java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #20            //  "Hello"
   #4 = Methodref          #21.#22        //  java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #23            //  org/jvminternals/SimpleClass
   #6 = Class              #24            //  java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lorg/jvminternals/SimpleClass;
  #14 = Utf8               sayHello
  #15 = Utf8               SourceFile
  #16 = Utf8               SimpleClass.java
  #17 = NameAndType        #7:#8          //  "<init>":()V
  #18 = Class              #25            //  java/lang/System
  #19 = NameAndType        #26:#27        //  out:Ljava/io/PrintStream;
  #20 = Utf8               Hello
  #21 = Class              #28            //  java/io/PrintStream
  #22 = NameAndType        #29:#30        //  println:(Ljava/lang/String;)V
  #23 = Utf8               org/jvminternals/SimpleClass
  #24 = Utf8               java/lang/Object
  #25 = Utf8               java/lang/System
  #26 = Utf8               out
  #27 = Utf8               Ljava/io/PrintStream;
  #28 = Utf8               java/io/PrintStream
  #29 = Utf8               println
  #30 = Utf8               (Ljava/lang/String;)V
{
  public org.jvminternals.SimpleClass();
    Signature: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
        0: aload_0
        1: invokespecial #1    // Method java/lang/Object."<init>":()V
        4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
          0      5      0    this   Lorg/jvminternals/SimpleClass;

  public void sayHello();
    Signature: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
        0: getstatic      #2    // Field java/lang/System.out:Ljava/io/PrintStream;
        3: ldc            #3    // String "Hello"
        5: invokevirtual  #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        8: return
      LineNumberTable:
        line 6: 0
        line 7: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
          0      9      0    this   Lorg/jvminternals/SimpleClass;
}
```

클래스 파일은 상수 풀, 생성자, sayHello 메서드 이렇게 3가지 주요 부분을 보여준다.

- 상수 풀: 심볼 테이블이 제공하는 정보와 동일한 정보가 제공되며 자세한 내용은 [런타임 상수 풀]()에서 다룬다.
- 메서드: 4 가지 주요 영역에 대한 정보
    - 시그너처(signature)와 접근 플래그(access flag)
    - 바이트코드
    - LineNumberTable: 바이트코드 명령어가 소스 코드의 몇 번째 행에 대응되는지를 보여주며 이 정보는 디버거에게 제공된다. 예를 들어 sayHello 메서드의 경우 자바 코드에서 6행은 바이트코드의 0번 위치에, 7행은 8번 위치에 대응된다.
    - LocalVariableTable: 프레임에 담긴 모든 로컬 변수 목록이 저장된다. 위 두 사례(생성자, sayHello 메서드)에서는 모두 this가 유일한 로컬 변수다.

바이트코드에는 다음과 같은 opcode가 사용되었다.

opcode | 설명
--- | ---
aload_0 | `aload_<n>`의 형식으로 구성되는 opcode 그룹의 하나로서 객체 레퍼런스를 오퍼랜드 스택에 로딩한다. `<n>`은 로컬 변수 배열에서의 위치를 참조하는데, 0, 1, 2, 3만 값으로 가질 수 있다. 비슷한 형식으로 값을 로딩하는 opcode로 `iload_<n>`, `lload_<n>`, `fload_<n>`, `dload_<n>`이 있다. i, l, f, d는 각각 int, long, float, double을 의미한다. 로컬 변수 배열에서의 인덱스가 3보다 큰 로컬 변수는 `iload`, `lload`, `fload`, `dload`를 사용해서 로딩할 수 있다. Xload 류의 opcode는 모두 하나의 오퍼랜드(로딩할 로컬 변수의 로컬 변수 배열에서의 인덱스)만을 받는다.
ldc | 런타임 상수 풀에 있는 상수를 오퍼랜드 스택에 push 한다.
getstatic | 런타임 상수 풀에 있는 정적 필드에 있는 정적 값을 오퍼랜드 스택에 push 한다.
invokdespecial, invokevirtual | 메서드를 호출하는 opcode의 하나. 메서드를 호출하는 opcode는 `invokedynamic`, `invokeinterface`, `invokespecial`, `invokestatic`, `invokevirtual`이 있다. `invokevirtual`은 객체의 클래스를 기반으로 메서드를 호출하고, `invokespecial`은 초기화 메서드(생성자)나 private 메서드, 수퍼클래스의 메서드를 호출할 때 사용된다.
return | 값을 반환하며 `ireturn`, `lreturn`, `freturn`, `dreturn`과 `areturn`도 있다. i, l, f, d는 각각 int, long, float, double을 의미하며 `areturn`은 객체 참조를 반환한다. `return`은 `void`를 반환한다.

일반적인 바이트코드에서 대부분의 opcode는 로컬 변수, 오퍼랜드 스택, 런타임 상수 풀과 상호작용한다.

SimpleClass의 생성자는 두 개의 명령어로 구성되는데 첫 번째 명령어는 `this`를 오퍼랜드 스택에 push하고, 두 번째 명령어는 수퍼클래스의 생성자를 호출하는데 이 때 오퍼랜드 스택에 있던 `this`를 꺼내어 사용한다.

![Imgur](https://i.imgur.com/M0TYrpl.png)

sayHello() 메서드는 런타임 상수 풀을 사용해서 심볼릭 참조를 실제 참조로 해석해야하기 때문에 앞에서 다룬 생성자보다 더 복잡하다. 첫 번째 opcode인 `getstatic`은 `System` 클래스의 정적 필드인 `out`에 대한 참조를 오퍼랜드 스택에 push 한다. 두 번째 opcode인 `ldc`는 문자열 "Hello"를 오퍼랜드에 push 한다. 마지막 opcode인 `invokevirtual`은 `System.out`의 `println` 메서드를 호출하면서 오퍼랜드 스택에서 "Hello"를 꺼내서 인자로 사용하고 `println` 메서드를 위한 새 프레임을 현재 실행 중인 스레드 내에 생성한다.

![Imgur](https://i.imgur.com/uZdtM0p.png)

### 클래스로더

JVM은 부트스트랩 클래스로더(bootstrap classloader)를 사용해서 초기 클래스를 로딩하는 것으로 시작된다. 이 초기 클래스는 `public static void main(String[])`이 호출되기 전에 링크되고 초기화 된다. `public static void main(String[])` 메서드가 실행되면 추가적으로 필요한 클래스와 인터페이스의 로딩, 링킹(linking), 초기화(initianlization)가 진행된다.

**`로딩(Loading)`**은 특정한 이름을 가진 클래스나 인터페이스를 나타내는 클래스 파일을 찾아서 읽고 바이트 배열에 담는 과정을 의미한다. 바이트 배열에 담긴 정보는 파싱되어 해당 정보가 `Class` 객체를 나타내는지, 올바른 메이저, 마이너 버전 정보를 포함하고 있는지 확인하는데 사용된다. 이 과정에서 수퍼클래스로 지정된 클래스나 인터페이스도 모두 함께 로딩된다. 이 로딩 과정이 완료되면 바이너리로 표현된 클래스나 인터페이스가 객체로 생성된다.

**`링킹(Linking)`**은 클래스나 인터페이스, 필요한 경우 직접적인 수퍼클래스나 수퍼인터페이스, 배열인 경우 원소 타입을 확인하고 준비하는 과정을 의미한다. Linking은 필수인 확인(verifying), 준비(preparing) 과정과 옵션인 해석(resolving) 과정으로 구성된다.

- **`확인(Verifying)`**은 클래스나 인터페이스의 바이너리 표현이 구조적으로 올바르고 자바 프로그래밍 언어와 JVM의 문법적 요구사항을 준수하는지 확인하는 과정이다. 예를 들어 다음과 같은 검사가 수행된다.

    1. 형식화된(formatted) 심볼 테이블의 일관성 및 정확성
    1. final 메서드나 클래스의 재정의(override) 여부
    1. 메서드의 접근 제어 키워드 준수 여부
    1. 메서드 파라미터의 갯수와 타입의 정확성
    1. 바이트코드가 스택을 잘못된 방식으로 조작하는지 여부
    1. 변수가 사용되기 전 초기화
    1. 변수의 값과 타입의 정확성

확인 단계에서 이런 감사가 수행되는 것은 검사를 반드시 런타임에 수행할 필요가 없음을 의미한다. 링킹 단계에서 확인 작업이 수행되면 클래스 로딩 속도를 느리게 만들지만, 바이트코드를 실행할 때는 이 검사를 수행하지 않아도 되므로 실행 속도는 높일 수 있다.

- **`준비(Preparing)`**는 메서드 테이블처럼 JVM이 사용하는 정적 저장공간이나 자료 구조를 위한 메모리 할당과 관련이 있다. 정적 필드는 준비 단계에서 생성되고 기본값으로 초기화된다. 하지만 생성자 코드나 그 외 어떤 코드도 준비 단계에서는 실행되지 않는다.

- **`해석(Resoving)`**은 심볼릭 참조가 가리키는 클래스나 인터페이스를 로딩해서 참조가 올바른지 검사하는 과정이다. 해석은 링킹 과정에서 필수적으로 수행되어야 하는 과정은 아니다. 해석 과정이 링킹 단계에서 수행되지 않으면 심볼릭 참조가 가리키는 클래스나 인터페이스가 실제로 처음 사용되는 시점에 해석 과정이 수행된다.

**`초기화(Initialization)`**는 클래스나 인터페이스의 초기화 메서드(`<clinit>`)를 실행해서 클래스나 인터페이스를 초기화하는 과정이다.

![Imgur](https://i.imgur.com/NtAwixB.png)

JVM에는 서로 다른 역할을 담당하는 여러 개의 클래스로더가 있다. 최상위 클래스로더인 부트스트랩 클래스로더를 제외한 모든 클래스로더는 자기 자신을 로드한 부모 클래스로더에게 클래스로딩을 위임한다.

- **`부트스트랩 클래스로더(Bootstrap Classloader)`**는 JVM이 로딩될 때 인스턴스화 되기 때문에 일반적으로 네이티브 코드로 구현된다. 부트스트랩 클래스로더는 예를 들어 rt.jar 처럼 기본적인 자바 API에 사용되는 클래스나 인터페이스의 로딩을 담당한다. 부트스트랩 클래스로더는 부트 클래스패스에 있는 신뢰도가 높은 클래스만을 로딩하기 때문에 일반적인 클래스 로딩 시 수행하는 많은 검증(validation) 과정을 생략한다.

- **`확장 클래스로더(Extension Classloader)`**는 보안 관련 확장 같은 표준 Java extension API에 사용되는 클래스나 인터페이스를 로딩한다.(옮긴이: Java 9부터는 Platform Classloader로 이름이 바뀌었다.)

- **`애플리케이션 클래스로더(Application Classloader)`**는 애플리케이션 기본 클래스로더로서 클래스패스에 있는 애플리케이션 클래스나 인터페이스를 로딩한다.

- **`사용자 정의 클래스로더(User Defined Classloader)`**도 애플리케이션 클래스나 인터페이스를 로딩하기 위해 사용될 수 있다. 사용자 정의 클래스로더는 런타임 리로딩(reloading)이나 톰캣 같은 웹서버의 필요에 의해 다른 그룹으로 분류되어야 하는 클래스나 인터페이스의 로딩 같은 특수 목적의 클래스로딩에 사용된다.

![Imgur](https://i.imgur.com/kDAmqku.png)

### 고속 클래스로딩

클래스 데이터 공유(Class Data Sharing, CDS) 기능이 HotSpot JVM 버전 5부터 도입되었다. JVM의 설치 과정에서 설치 프로그램은 rt.jar 같은 몇 가지 핵심 JVM 클래스를 메모리 맵 공유 아카이브(memory-mapped shared archive)에 로딩한다. CDS는 클래스로딩 시간을 단축시켜서 JVM 시작 속도를 개선하고 핵심 JVM 클래스가 서로 다른 JVM 인스턴스 사이에도 공유될 수 있도록 해서 메모리 사용량도 줄일 수 있다.

### 메서드 영역의 위치

Java SE 7의 JVM 스펙에 다음과 같이 명시되어 있다.

>메서드 영역은 논리적으로 힙의 일부지만, 간단한 구현체에서는 메서드 영역은 가비지 컬렉션이나 압축(compact)에서 제외할 수 있다.

하지만 이와 반대로 오라클 JVM 용 jconsole에서는 메서드 영역(코드 캐시 포함)이 Non-Heap 영역에 포함되어있는 것을 볼 수 있다. OpenJDK 코드에서는 VM의 코드 캐시가 ObjectHeap와 분리된 필드로 되어 있다.

### 클래스로더 참조

로딩된 모든 클래스는 자기 자신을 로딩한 클래스로더에 대한 참조를 가지고 있다. 클래스로더도 자기가 로딩한 모든 클래스에 대한 참조를 가지고 있다.

### 런타임 상수 풀

JVM은 타입별로 런타임 상수 풀을 관리한다. 런타임 상수 풀은 심볼 테이블과 유사하며 심볼 테이블보다는 더 많은 데이터를 포함한다. 자바 바이트코드가 필요로하는 데이터가 직접 바이트코드로 저장하기에 너무 크면 런타임 상수 풀에 저장되고, 바이트코드는 런타임 상수 풀에 저장된 데이터에 대한 참조를 보유한다. 런타임 상수 풀은 [앞에서 설명]()한 것처럼 동적 링크에 사용된다.

런타임 상수 풀에 저장되는 데이터는 다음과 같다.

- 숫자 리터럴
- 문자열 리터럴
- 클래스 참조
- 필드 참조
- 메서드 참조

예를 들어 다음 코드는

```java
Object foo = new Object();
```

다음과 같은 바이트코드로 컴파일된다.

```java
 0:     new #2              // Class java/lang/Object
 1:     dup
 2:     invokespecial #3    // Method java/ lang/Object "<init>"( ) V
```

`new` opcode 다음에는 `#2`라는 오퍼랜드가 오는데, 이 오퍼랜드는 런타임 상수 풀의 인덱스다. `#2`는 런타임 상수 풀의 두 번째 아이템을 가리킨다. 두 번째 아이템은 클래스 참조이며 이 아이템은 다시 런타임 상수 풀에 UTF-8 문자열로 저장된 클래스의 이름(`// Class java/lang/Object`) 아이템을 가리킨다. 이 심볼릭 링크는 java.lang.Object 클래스를 찾는데 사용된다. `new` opcode는 클래스 인스턴스를 생성하고 값을 초기화한다. 새로 생성된 클래스에 대한 참조는 오퍼랜드 스택에 추가된다. `dup` opcode는 오퍼랜드 스택의 맨 위에 있는 참조에 대한 복사본을 만들어서 오퍼랜드 스택의 맨 위에 추가한다. 마지막으로 인스턴스 초기화 메서드가 `invokespecial`에 의해 호출된다. 이 오퍼랜드는 런타임 상수 풀에 대한 참조를 포함하고 있다. 초기화 메서드는 오퍼랜드 스택의 맨 위에 있는 값을 꺼내서 초기화 메서드의 인자로 사용한다. 오퍼랜드 스택에는 방금 새로 생성되어 초기화 된 객체를 가리키는 참조만 남는다.

아래와 같은 간단한 클래스를 컴파일하면,

```java
package org.jvminternals;

public class SimpleClass {

    public void sayHello() {
        System.out.println("Hello");
    }

}
```

생성되는 클래스 파일 안에 다음과 같은 런타임 상수 풀이 만들어진다.

```java
Constant pool:
   #1 = Methodref          #6.#17         //  java/lang/Object."<init>":()V
   #2 = Fieldref           #18.#19        //  java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #20            //  "Hello"
   #4 = Methodref          #21.#22        //  java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #23            //  org/jvminternals/SimpleClass
   #6 = Class              #24            //  java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lorg/jvminternals/SimpleClass;
  #14 = Utf8               sayHello
  #15 = Utf8               SourceFile
  #16 = Utf8               SimpleClass.java
  #17 = NameAndType        #7:#8          //  "<init>":()V
  #18 = Class              #25            //  java/lang/System
  #19 = NameAndType        #26:#27        //  out:Ljava/io/PrintStream;
  #20 = Utf8               Hello
  #21 = Class              #28            //  java/io/PrintStream
  #22 = NameAndType        #29:#30        //  println:(Ljava/lang/String;)V
  #23 = Utf8               org/jvminternals/SimpleClass
  #24 = Utf8               java/lang/Object
  #25 = Utf8               java/lang/System
  #26 = Utf8               out
  #27 = Utf8               Ljava/io/PrintStream;
  #28 = Utf8               java/io/PrintStream
  #29 = Utf8               println
  #30 = Utf8               (Ljava/lang/String;)V
```

런타임 상수 풀은 다음과 같은 타입을 포함하고 있다.

타입 이름 | 설명
--- | ---
Integer | 4바이트 int 상수
Long | 8바이트 long 상수
Float | 4바이트 float 상수
Double | 8바이트 double 상수
String | 런타임 상수 풀에 실제 바이트로 저장된 UTF8 아이템을 가리키는 문자열 상수
Utf8 | UTF8로 인코딩된 문자열을 나타내는 바이트 스트림
Class | 런타임 상수 풀에 Utf8 타입으로 저장된 클래스 전체 이름 문자열(동적 링크에 사용됨)을 가리키는 클래스 상수
NameAndType | 런타임 상수 풀에 Utf8 타입으로 저장된 메서드나 필드의 이름과 그 메서드나 필드의 타입을 콜론(:)으로 구분해서 표현
Fieldref, Methodref, InterfaceMethodref | 런타임 상수 풀에 저장된 Class 아이템과 NameAndType 아이템을 점(.)으로 구분해서 표현

### 예외 테이블

예외 테이블(Exception Table)은 예외 핸들러별로 다음과 같은 정보를 저장한다.

- 시작 지점(Start point)
- 종료 지점(End point)
- 핸들러 코드 기준 PC 오프셋
- 잡혀진 예외 클래스를 가리키는 런타임 상수 풀 인덱스

메서드에 `try-catch`나 `try-finally` 예외 핸들러가 정의되면 예외 테이블이 생성된다. 예외 테이블에는 예외 핸들러나 `finally` 블록에 대한 정보(처리하는 예외의 타입, 예외 처리 코드의 위치 등)가 저장된다.

예외가 던져지면 JVM은 현재 실행 중인 메서드 안에서 해당 예외에 매칭되는 핸들러를 찾는다. 핸들러가 발견되지 않으면 현재 실행 중인 메서드는 비정상 종료 처리되고 현재 실행 중인 메서드의 프레임이 스택에서 꺼내어지고 처리되지 않은 예외는 현재 실행 중인 메서드를 호출한 메서드로 다시 던져진다. 모든 프레임이 스택에서 꺼내어질 때까지 해당 예외에 매칭되는 핸들러가 발견되지 않으면 현재 실행 중인 스레드는 종료된다. 핸들러가 발견되지 않는 예외가 main 스레드 같은 마지막 non-daemon 스레드에서 던져지면 JVM 자체도 종료된다.

`finally` 예외 핸들러는 모든 타입의 예외에 매칭되는 핸들러로서, 어떤 예외가 발생하더라도 `finally` 예외 핸들러는 무조건 실행된다. 아무런 예외가 발생하지 않더라도 return 문이 실행되기 전에 `finally` 블록으로 점프되어 `finally` 블록 내용이 무조건 실행된다.

### 심볼 테이블

타입 별로 존재하는 런타임 상수 풀에 더해서 JVM은 Permanent Generation 영역에 심볼 테이블을 보유한다. 심볼 테이블은 심볼 포인터를 심볼에 매핑하는 해시테이블(즉, Hashtable<Symbol\*, Symbol>)이며, 각 클래스의 런타임 상수 풀에 저장된 모든 심볼에 대한 포인터를 포함하고 있다.

참조 카운팅(Reference counting)은 심볼 테이블에서 심볼이 제거되는 시점을 제어하는데 사용된다. 예를 들어 클래스 하나가 언로딩되면 런타임 상수 풀에서 이 클래스를 가리키던 모든 심볼에 대한 참조 카운트가 감소된다. 심볼 테이블에 있는 하나의 심볼에 대한 참조 카운트가 0이 되면 심볼 테이블은 이 심볼이 더 이상 참조되지 않는다는 사실을 알고 그 심볼을 심볼 테이블에서 언로드한다. 심볼 테이블의 모든 아이템은 정식 형식(canonicalized form)으로 저장되어 효율을 개선하고 각 아이템이 유일하게 하나씩만 존재하는 것을 보장한다.

### Interned String(String table)

자버 언어 스펙에 따르면 동일한 순서로 구성된 유니코드 포인트를 포함하고 있는 문자열 리터럴은 동일한 String 인스턴스를 참조해야 한다. `String.intern()` 메서드는 문자열 리터럴에 대한 참조를 반환하는데 다음의 등식은 참이다.

```java
("j" + "v" + "m").intern == "jvm"
```

HotSpot JVM 에서는 interned string은 문자열 테이블(String table)에 저장된다. 문자열 테이블은 객체를 심볼에 매핑하는 해시테이블(즉, `Hashtable<oop, Symbol>`)인데 Permanent Generation 안에 저장된다. 문자열 테이블의 모든 아이템은 정식 형식(canonicalized form)으로 저장되어 효율을 개선하고 각 아이템이 유일하게 하나씩만 존재하는 것을 보장한다.

문자열 리터럴은 해당 문자열 리터럴이 사용된 클래스가 로딩될 때 컴파일러에 의해 자동으로 interned 되어 심볼 테이블에 추가된다. String 클래스의 인스턴스는 `String.intern()`을 호출해서 명시적으로 interned 될 수도 있다. `String.intern()`은 해당 문자열이 문자열 테이블에 존재하면 그 문자열에 대한 참조를 반환하고, 문자열 테이블에 없으면 새로 추가하고 새로 추가한 문자열에 대한 참조를 반환한다.



