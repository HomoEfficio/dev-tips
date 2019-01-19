# Back to the Essence - Java 컴파일에서 실행까지 - (2)

Java 소스 코드가 어떻게 컴파일되고 실행되는지 살짝 깊게 알아보자.

# 실행

자바 애플리케이션은 `java` 명령어로 실행할 수 있다. [오라클의 Tools Reference 문서](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE)에 나오는 `java`에 대한 설명은 다음과 같다.

>`java` 명령어는 자바 애플리케이션을 시작한다.  
>`java` 명령어는 먼저 JRE(Java Runtime Environment)를 시작하고,  
>인자로 지정된 클래스(`public static void main(String[] args)`를 포함하고 있는 클래스)를 로딩하고,  
>`main()` 메서드를 호출한다.

## JDK, JRE, JVM

`java`는 JRE를 시작한다고 하니, JDK, JRE, JVM의 관계를 그림 한 장으로 가볍게 훑고 지나가자.

![Imgur](https://i.imgur.com/x5J0ZzB.png)

- JDK: 자바 개발 환경 - 컴파일러, 역어셈블러, 디버거, 의존관계분석 등 개발에 필요한 도구 제공
- JRE: 자바 실행 환경 - 자바 실행 명령, 클래스로더와 바이트코드의 실행에 필요한 기본 라이브러리 제공
- JVM: 자바 가상 머신 - 바이트코드 인터프리터, JIT 컴파일러, 링커, 명령어 세트, 가비지 컬렉터, 메모리 모델 등 OS에 독립적으로 실행될 수 있는 추상층 제공

대략 다음과 같이 정리할 수 있다.

>JDK를 사용해서 바이트코드(class 파일)를 만들고, 
>
>JRE를 사용해서 바이트코드를 실행하면,
>
>JVM이 기동되면서 바이트코드의 실질적인 실행(실제 OS에 메모리 할당/회수, 시스템 명령 호출 등을 요청)을 담당한다.


## JRE 시작

`java` 명령 실행에 의해 JRE가 시작된다는 것은 결국 `java` 명령어의 인자로 지정된 클래스를 실행하기 위한 자바 실행 환경이 조성됨을 의미한다. 

`java` 명령어의 인자로 지정한 설정 옵션에 맞게 JVM이 실행되고, JVM이 클래스로더를 이용해서 `initial class`를 `create`하고, `initial class`를 `link`하고, `initialize`하고, main 메서드를 호출한다.([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.2) 참고)


### 용어 정리

몇 가지 용어를 일부러 스펙에 나온 원어 그대로 썼는데 의미는 다음과 같다.

- initial class: JVM 구현에 따라 다를 수 있지만 일반적으로 main 메서드를 포함하는 클래스로서 java 명령어의 인자로 지정되는 클래스 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.2) 참고)
- create (a class or interface): 해당 클래스나 인터페이스의 바이트코드를 로딩해서 JVM이 할당한 메모리(Method Area, 메서드 영역)에 construction하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.3) 참고)
- link (a class or interface): 해당 클래스나 인터페이스의 바로 위 수퍼클래스나 수퍼인터페이스, 또는 배열일 경우 배열의 원소인 클래스나 인터페이스를 확인(verify)/준비(prepare)하고, 심볼릭 참조를 해석(resolve)하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4) 참고)
- initialize (a class or interface): 해당 클래스나 인터페이스의 initialization method를 실행하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.5) 참고)

앞으로 **initial class는 시작 클래스**, **create은 생성**, **link는 링크**, **initialize는 초기화** 라고 쓴다. 한 가지 유의할 것은 여기서 말하는 **생성(create)은 JVM의 힙(heap)에 객체를 생성하는 것만을 지칭하는 것이 아니라 JVM의 메모리 어딘가에 정보를 생성하는 것을 모두 지칭** 한다.


## 런타임 데이터 영역

`java` 명령어로 자바 애플리케이션을 실행하면 JVM이 실행되면서 시작 클래스를 생성, 링크, 초기화하고 main 메서드를 호출한다고 했다. 시작 클래스를 create 한다는 것은 시작 클래스의 바이트코드를 읽어서 JVM의 메모리 어딘가에 쓰는 것을 의미한다. JVM의 메모리는 어떻게 생겼을까?

JVM은 프로그램의 실행에 사용되는 메모리를 런타임 데이터 영역(Runtime Data Area)이라고 부르는 몇 가지 영역으로 나눠서 관리한다. 스펙의 목차로 보면 밋밋하게 다음과 같이 나열되어 있다.

1. PC 레지스터
1. JVM 스택
1. 힙(Heap)
1. 메서드 영역(Method Area)
1. 런타임 상수 풀(Run-Time Constnat Pool)
1. 네이티브 메서드 스택

이렇게 보면 위 6가지가 동등한 최상위 수준에서 분류되는 것처럼 보인다. 하지만, 실제 스펙의 설명을 보면 다음과 같이 구분하는 것이 더 적합하다.

![Imgur](https://i.imgur.com/Mh4DuRB.png)

여기서 '단위'라는 구분 단계를 추가한 이유는 스펙에도 `per-class`, `per-thread` 라는 표현이 나오기 때문인데, 여기에서의 '단위'는 생명 주기와 생성 단위를 의미한다.

JVM 단위에 속하는 **힙과 메서드 영역은 JVM이 시작될 때 생성되고, JVM이 종료될 때 소멸되며, JVM 하나에 힙 하나, 메서드 영역도 하나가 생성** 된다.

마찬가지로 클래스 단위에 속하는 **런타임 상수 풀은 클래스가 생성/소멸될 때 함께 생성/소멸되며, 클래스 하나에 런타임 상수 풀도 하나가 생성** 된다.

스레드 단위에 속하는 **PC 레지스터, JVM 스택, 네이티브 메서드 스택도 스레드가 생성/소멸될 때 함께 생성/소멸되며, 스레드 하나에 PC 레지스터, JVM 스택, 네이티브 메서드 스택도 각 하나씩 생성** 된다.

자. 이제 6가지 영역을 좀 더 자세히 알아보자.

라고 진행하면 너무 뻔한 나열식이라 머리에 잘 안 남는다. 그러니 다음과 같이 간단한 예제 코드 실행 과정과 함께 살펴보자.


# 예제 코드 실행

예제 자바 코드 및 바이트코드 관련 내용은 스펙이 아니라 JVM Internals라는 글([원문](https://blog.jamesdbloom.com/JVMInternals.html), [번역](https://github.com/HomoEfficio/dev-tips/blob/master/JVM%20Internals.md))에서 따왔다.

## 예제 코드

흔히 보는 헬로 월드 프로그램의 소스 코드다.

```java
package org.jvminternals;

public class SimpleClass {

    public void sayHello() {
        System.out.println("Hello");
    }

}
```

## 컴파일된 바이트코드

컴파일된 바이트코드는 다음과 같다.

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

앞에서 JDK, JRE, JVM 관계로 설명했지만 위와 같은 바이트코드를 만드는 과정까지는 JDK에서 담당한다.

앞에서 자바 프로그램이 실행되면 다음과 같이 전개된다고 설명했다. 

>**JVM이 실행**되고, JVM이 클래스로더를 이용해서 **시작 클래스를 생성**하고, **링크**하고, **초기화**하고, **main 메서드를 호출**한다.

이제 `java org.jvminternals.SimpleClass` 명령을 실행하면 어떻게 진행되는지 차근차근 살펴보자.


## JVM 실행

JVM이 실행되면 JVM 단위로 생성되는 힙과 메서드 영역이 함께 생성된다.

![Imgur](https://i.imgur.com/KXJsPgs.png)

### 힙

**힙은 인스턴스화 된 모든 클래스 인스턴스와 배열을 저장하는 공간**이며, **모든 JVM 스레드에 공유**된다.

힙에 저장된 객체에 할당된 메모리는 명시적인 방법으로는 절대 회수되지 못하며, 오직 가비지 컬렉터(garbage collector)에 의해서만 회수될 수 있다.

SimpleClass는 이 시점에서는 아직 인스턴스화 되지 않았으므로 힙은 비어있다.

### 메서드 영역

**메서드 영역은 런타임 상수 풀, 필드와 메서드 데이터, 생성자 및 메서드의 코드 내용을 저장**한다. 저장되는 내용은 위에서 살펴봤던 바이트코드의 내용과 거의 일치한다. 거의라고 얘기하는 이유는 바이트코드에는 런타임 상수 풀이 아니라 그냥 상수 풀(constanta pool)이 포함되어 있기 때문이다. 런타임 상수 풀은 이 상수 풀을 바탕으로 런타임에, 더 구체적으로는 메서드 영역에 저장될 때 만들어진다.

그래서 엄밀히 말하면 정확하지 않지만, **바이트코드 내용이 메서드 영역에 저장된다**라고 이해해도 크게 틀리지는 않다.

SimpleClass는 이 시점에서는 아직 생성되지 않았으므로 메서드 영역도 비어있다.


## 시작 클래스 생성

시작 클래스는 SimpleClass를 지칭하며 시작 클래스를 생성하는 것은 파일시스템에 있는 SimpleClass.class 파일을 JVM의 메서드 영역으로 읽어들이는 것을 말한다고 했다. 따라서 이 시점에서 SimpleClass의 바이트코드 내용이 메서드 영역에 저장된다.

![Imgur](https://i.imgur.com/0CTTUDC.png)


### 런타임 상수 풀

클래스가 생성되면 런타임 상수 풀도 함께 생성된다고 했다. 런타임 상수 풀에는 컴파일 타임에 이미 알 수 있는 숫자 리터럴 값부터 런타임에 해석되는 메서드와 필드의 참조까지를 포괄하는 여러 종류의 상수가 포함된다. 런타임 상수 풀은 다른 전통적인 언어에서 말하는 심볼 테이블과 비슷한 기능을 한다고 보면 된다.

![Imgur](https://i.imgur.com/fz81deC.png)

###



