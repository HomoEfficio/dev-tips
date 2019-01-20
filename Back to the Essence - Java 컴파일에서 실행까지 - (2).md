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

몇 가지 용어를 일부러 스펙에 나온 원어 그대로 썼는데 스펙상의 의미는 다음과 같다.

- initial class: JVM 구현에 따라 다를 수 있지만 일반적으로 main 메서드를 포함하는 클래스로서 java 명령어의 인자로 지정되는 클래스 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.2) 참고)
- create (a class or interface): 해당 클래스나 인터페이스의 바이트코드를 로딩해서 JVM이 할당한 메모리(Method Area, 메서드 영역)에 construction하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.3) 참고)
- link (a class or interface): 해당 클래스나 인터페이스의 바로 위 수퍼클래스나 수퍼인터페이스, 또는 배열일 경우 배열의 원소인 클래스나 인터페이스를 확인(verify)/준비(prepare)하고, 심볼릭 참조를 해석(resolve)해서 JVM에서 실행될 수 있는 상태로 만드는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4) 참고)
- initialize (a class or interface): 해당 클래스나 인터페이스의 initialization method를 실행하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.5) 참고)

위 설명에는 없지만 중요한 용어인 로딩의 스펙상의 의미는 다음과 같다.

- load (a class or interface): 해당 클래스나 인터페이스의 바이너리 표현을 찾아서 그 바이너리 표현으로부터 클래스나 인터페이스를 생성(create)하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html) 참고)

앞으로 **initial class는 시작 클래스**, **create은 생성**, **link는 링크**, **initialize는 초기화**, **load는 로딩**이라고 쓴다. 한 가지 유의할 것은 여기서 말하는 **생성(create)은 JVM의 힙(heap)에 객체를 생성하는 것만을 지칭하는 것이 아니라 JVM의 메모리 어딘가에 자료구조를 생성하는 것을 모두 지칭**한다.


## 런타임 데이터 영역

`java` 명령어로 자바 애플리케이션을 실행하면 JVM이 실행되면서 시작 클래스를 생성, 링크, 초기화하고 main 메서드를 호출한다고 했다. 시작 클래스를 생성한다는 것은 시작 클래스의 바이트코드를 읽어서 JVM의 메모리 어딘가에 쓰는 것을 의미한다. JVM의 메모리는 어떻게 생겼을까?

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


## 예제 코드

### 자바 소스 코드

흔히 보는 헬로 월드 프로그램의 소스 코드다.

```java
package homo.efficio.jvm.sample;

public class SimpleClass {

    public static void main(String[] args) {
        System.out.println("Hello, JVM");
    }
}


```

### 컴파일된 바이트코드

컴파일된 바이트코드는 다음과 같다.

```java
Classfile /C:/gitrepo/scratchpad/plain-java-scratchpad/out/production/classes/homo/efficio/jvm/sample/SimpleClass.class
  Last modified 2019. 1. 20.; size 583 bytes
  MD5 checksum a5fcf145be8e22dfcf53556a410ace9f
  Compiled from "SimpleClass.java"
public class homo.efficio.jvm.sample.SimpleClass
  minor version: 0
  major version: 53
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #5                          // homo/efficio/jvm/sample/SimpleClass
  super_class: #6                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // Hello, JVM
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // homo/efficio/jvm/sample/SimpleClass
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lhomo/efficio/jvm/sample/SimpleClass;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               SimpleClass.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               Hello, JVM
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               homo/efficio/jvm/sample/SimpleClass
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
{
  public homo.efficio.jvm.sample.SimpleClass();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lhomo/efficio/jvm/sample/SimpleClass;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello, JVM
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 10: 0
        line 11: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
}
SourceFile: "SimpleClass.java"

```

앞에서 JDK, JRE, JVM 관계로 설명했지만 위와 같은 바이트코드를 만드는 과정까지는 JDK에서 담당한다.

앞에서 자바 프로그램이 실행되면 다음과 같이 전개된다고 설명했다. 

>**JVM이 실행**되고, JVM이 클래스로더를 이용해서 **시작 클래스를 생성**하고, **링크**하고, **초기화**하고, **main 메서드를 호출**한다.

이제 `java org.jvminternals.SimpleClass` 명령을 실행하면 어떻게 진행되는지 차근차근 살펴보자.


## JVM 실행

JVM이 실행되면 JVM 단위로 생성되는 힙과 메서드 영역이 함께 생성된다.

### 힙

**힙은 인스턴스화 된 모든 클래스 인스턴스와 배열을 저장하는 공간**이며, **모든 JVM 스레드에 공유**된다.

힙에 저장된 객체에 할당된 메모리는 명시적인 방법으로는 절대 회수되지 못하며, 오직 가비지 컬렉터(garbage collector)에 의해서만 회수될 수 있다.

SimpleClass는 이 시점에서는 아직 인스턴스화 되지 않았으므로 힙은 비어있다.

### 메서드 영역

**메서드 영역은 런타임 상수 풀, 필드와 메서드 데이터, 생성자 및 메서드의 코드 내용을 저장**한다. 저장되는 내용은 위에서 살펴봤던 바이트코드의 내용과 거의 일치한다. 거의라고 얘기하는 이유는 바이트코드에는 런타임 상수 풀이 아니라 그냥 상수 풀(constanta pool)이 포함되어 있기 때문이다. 런타임 상수 풀은 이 상수 풀을 바탕으로 런타임에, 더 구체적으로는 메서드 영역에 저장될 때 만들어진다.

그래서 엄밀히 말하면 정확하지 않지만, **바이트코드 내용이 메서드 영역에 저장된다**라고 이해해도 크게 틀리지는 않다.

SimpleClass는 이 시점에서는 아직 생성되지 않았으므로 메서드 영역도 비어있다.

![Imgur](https://i.imgur.com/KXJsPgs.png)


## 시작 클래스 생성

시작 클래스는 SimpleClass를 지칭하며 시작 클래스를 생성하는 것은 파일시스템에 있는 SimpleClass.class 파일을 JVM의 메서드 영역으로 읽어들이는 것을 말한다고 했다. 따라서 **이 시점에서 SimpleClass의 바이트코드 내용이 메서드 영역에 저장**된다.

![Imgur](https://i.imgur.com/0CTTUDC.png)


### 런타임 상수 풀

클래스가 생성되면 런타임 상수 풀도 함께 생성된다고 했다. **런타임 상수 풀에는 컴파일 타임에 이미 알 수 있는 숫자 리터럴 값부터 런타임에 해석되는 메서드와 필드의 참조까지를 포괄하는 여러 종류의 상수가 포함**된다. 런타임 상수 풀은 다른 전통적인 언어에서 말하는 심볼 테이블과 비슷한 기능을 한다고 보면 된다.

![Imgur](https://i.imgur.com/fz81deC.png)


## 링크

**링크는 클래스나 인터페이스의 바로 위 수퍼클래스나 수퍼인터페이스, 또는 배열일 경우 배열의 원소인 클래스나 인터페이스를 확인(verify)/준비(prepare)하고, 심볼릭 참조를 해석(resolve)하는 과정**을 말한다.

그럼 확인, 준비, 해석은 뭘 의미하는 걸까?

### 확인

**확인(verification)은 클래스나 인터페이스의 바이너리 표현이 구조적으로 올바른지를 보장해주는 과정**이다. 확인 과정은 다른 클래스나 인터페이스의 로딩을 유발할 수도 있지만, 로딩된 다른 클래스나 인터페이스의 확인이나 준비를 필수적으로 유발하지는 않는다.

SimpleClass.class 파일은 정상적으로 컴파일되었으므로 구조적으로 올바르다고 가정하면, 확인 과정에서 SimpleClass의 부모 클래스인 Object
가 로딩된다.

![Imgur](https://i.imgur.com/yKf0zIp.png)

### 준비

**준비(preperation)는 클래스나 인터페이스의 정적 필드를 생성하고 기본값으로 초기화하는 과정**이다. 준비 과정에서는 JVM 코드의 실행을 필요로 하지 않으며, **기본값이 아닌 특정값으로 정적 필드를 초기화하는 과정은 준비 과정이 아니라 초기화 과정에서 수행**된다.

스펙에 정의된 [기본형 타입의 기본값](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-2.html#jvms-2.3)과 [참조형 타입의 기본값](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-2.html#jvms-2.4)은 다음과 같다.

타입 | 기본값
--- | ---
byte, short, int, long | 0
char | null(`'\u0000'`)
float, double | 0(positive zero)
boolean | false
참조형 | null


SimpleClass에는 정적 필드가 없으므로 이 과정에서 특별히 수행되는 것은 없다.

### 해석

**해석(resolution)은 런타임 상수 풀에 있는 심볼릭 참조가 구체적인 값을 가리키도록 동적으로 결정하는 과정**이다. 초기 상태의 런타임 상수 풀에 있는 심볼릭 참조는 해석되어져 있지 않다.

### 링크의 조건

JVM 스펙에서는 **링크가 언제 수행되어야 하는지는 규정하지 않고 유연하게 구현될 수 있는 여지**를 주고 있다. 단 다음과 같은 조건을 만족해야 한다.

- 클래스나 인터페이스는 링크되기 전에 먼저 완전히 로딩되어야 한다.
- 클래스나 인스턴스는 초기화되기 전에 먼저 완전히 확인되고 준비되어야 한다.
- 링크 관련 에러는 해당 클래스나 인터페이스에 대한 링크를 필요로 하는 행위가 수행되는 시점에 throw 되어야 한다.
- 동적으로 계산되는(dynamically-computed) 상수 A에 대한 심볼릭 참조는, A를 참조하는 `ldc`, `ldc_w`, `ldc2_w` 명령어가 실행되거나 A를 정적 인자로 참조하는 부트스트랩 메서드가 호출되기 전까지는 해석되지 않는다.
- 동적으로 계산되는(dynamically-computed) call site B에 대한 심볼릭 참조는, B를 정적 인자로 참조하는 부트스트랩 메서드가 호출되기 전까지는 해석되지 않는다.

일반적으로 만족되어야 하는 것은 1, 2, 3번째 조건이고 4, 5번째는 특수한 경우에 대한 조건이다.

해석 시점은 JVM 구현체에 따라 다를 수 있다. **지연(lazy) 링크 전략을 사용하면 클래스나 인터페이스에 포함된 심볼릭 참조는 해당 참조가 실제 사용될 때 개별적으로 해석**된다. 반면에 **즉시(eager) 링크 전략을 사용하면 클래스나 인터페이스가 확인될 때 모든 심볼릭 참조가 한꺼번에 해석**된다. 지연 링크를 사용하면 해석 과정은 클래스나 인터페이스가 초기화 된 후에 실행될 수도 있다.

링크 과정을 정리하면 다음과 같다.

>링크는 **확인, 준비, 해석 단계로 구성된다.**
>
>**클래스나 인터페이스는 완전히 로딩된 후에 확인과 준비가 수행돼야 하고, 완전히 확인되고 준비된 뒤에 초기화되어야 한다.**
>
>**해석은 초기화 이후에 실행될 수도 있다.**

스펙을 보면 해석은 다시 [클래스/인터페이스 해석](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4.3.1), [필드 해석](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4.3.2), [메서드 해석](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4.3.3), [인터페이스 메서드 해석](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4.3.4), [메서드 타입/핸들 해석](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4.3.5), [동적 계산 상수/콜사이트 해석](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4.3.6), 이렇게 6가지로 나눠서 자세한 설명이 나와 있으니 관심있다면 찾아보기로 하고 다시 예제로 돌아와 보자. 

SimpleClass의 상수 풀은 다음과 같았다.

```java
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // Hello, JVM
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // homo/efficio/jvm/sample/SimpleClass
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lhomo/efficio/jvm/sample/SimpleClass;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               SimpleClass.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               Hello, JVM
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               homo/efficio/jvm/sample/SimpleClass
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
```

설명의 편의를 위해 즉시 링크 방식으로 해석이 진행된다고 가정하고, 위 상수 풀에서 유도되는 런타임 상수 풀에 있는 심볼릭 참조의 해석 과정을 몇 개만 예로 살펴보자.

`#1 = Methodref          #6.#20         //  java/lang/Object."<init>":()V`

Object 클래스가 확인 과정에서 로딩되어 있으므로 메서드 영역에 저장된 Object 클래스의 바이트코드 내용에서 생성자(`<init>`)의 위치를 알아낼 수 있고, 그 위치를 `java/lang/Object."<init>"`의 값으로 해석할 수 있다.

![Imgur](https://i.imgur.com/us3J65x.png)

`#2 = Fieldref           #21.#22        //  java/lang/System.out:Ljava/io/PrintStream;`

System 클래스는 아직 로딩되어 있지 않으므로 먼저 로딩하고, 확인 후 준비 과정을 거치면서 System 클래스의 정적 필드인 `out`의 타입인 PrintStream 클래스도 로딩되고 `out`은 기본값인 null 로 초기화 된다.

![Imgur](https://i.imgur.com/VLz6GG8.png)

대략 이런 식으로 로딩-링크 과정이 연쇄적으로 수행되면서 메서드 영역이 채워지고, 메서드 영역 내에서 클래스 단위로 생성되는 런타임 상수 풀 안에 있는 심볼릭 참조가 가리키는 값들이 결정된다.

하지만 이것도 위에 썼 듯이 즉시 링크 방식일 때의 얘기고, **지연 링크를 사용했다면 각 클래스의 초기화가 수행된 이후에 해석 과정이 수행**될 수도 있다.

그럼 이제 초기화를 알아볼 차례다.


## 초기화

초기화(initialization)는 클래스나 인터페이스의 초기화 메서드(initialization method)를 실행할 때 수행되는 과정이다. 좀 쉽게 말하면 여기에서 말하는 초기화는 정적 초기화(static initialization)를 말한다고 볼 수 있다.

그럼 초기화 메서드는 무엇일까?

### 초기화 메서드

초기화 메서드(initialization method)는 두 가지가 있다.

#### 인스턴스 초기화 메서드

인스턴스 초기화 메서드는 자바 언어로 작성되는 생성자에 해당하며, 클래스는 0개 이상의 인스턴스 초기화 메서드를 가진다. 인스턴스 초기화 메서드는 다음의 조건을 충족한다.

- (인터페이스가 아니고) 클래스 안에 정의된다.
- (바이트코드 상에서) `<init>`라는 특수한 이름으로 표현된다.
- 반환 타입은 void

인스턴스 초기화 메서드는 생성자로서 힙에 인스턴스를 생성하는 역할을 담당하며, 여기에서 말하는 초기화와는 좀 다른 개념이다.

#### 클래스 초기화 메서드

앞에서 링크 과정의 준비 단계 설명에 초기화가 잠시 언급된 적이 있다.  
**정적 필드를 기본값으로 초기화 하는 것은 링크의 준비 단계에서 수행**되고, **특정값으로 초기화 하는 것은 초기화 단계에서 수행**된다고 했는데, 바로 이 클래스 초기화 메서드가 실행되면서 특정값으로의 초기화가 이루어진다.

클래스 또는 인터페이스 초기화 메서드는 다음의 조건을 충족한다.

- (바이트코드 상에서) `<clinit>`라는 특수한 이름으로 표현된다.
- 반환 타입은 void
- class 파일 버전 51 이상에서는 `ACC_STATIC` 플래그가 붙는다.

인스턴스 초기화 메서드는 생성자에 해당한다는 명확하고 직관적인 설명이 스펙에 있는데, 클래스 초기화 메서드는 아쉽게도 뭐에 해당하는지 스펙에는 구체적인 설명이 없다.

**클래스 초기화 메서드는 쉽게 말해 static 블록**이라고 볼 수 있다. 이건 말보다 코드가 훨씬 쉬우니 코드로 살펴보자.

```java
package homo.efficio.jvm.sample;

public class StaticInitSample {

    public final static int i;

    static {
        i = 123;
    }
}
```

컴파일한 후 `javap` 명령으로 바이트코드를 살펴보면 다음과 같다.

```java
public class homo.efficio.jvm.sample.StaticInitSample
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#17         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#18         // homo/efficio/jvm/sample/StaticInitSample.i:I
   #3 = Class              #19            // homo/efficio/jvm/sample/StaticInitSample
   #4 = Class              #20            // java/lang/Object
   #5 = Utf8               i
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lhomo/efficio/jvm/sample/StaticInitSample;
  #14 = Utf8               <clinit>
  #15 = Utf8               SourceFile
  #16 = Utf8               StaticInitSample.java
  #17 = NameAndType        #7:#8          // "<init>":()V
  #18 = NameAndType        #5:#6          // i:I
  #19 = Utf8               homo/efficio/jvm/sample/StaticInitSample
  #20 = Utf8               java/lang/Object
{
  public static final int i;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL

  public homo.efficio.jvm.sample.StaticInitSample();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lhomo/efficio/jvm/sample/StaticInitSample;

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: bipush        123
         2: putstatic     #2                  // Field i:I
         5: return
      LineNumberTable:
        line 8: 0
        line 9: 5
}
SourceFile: "StaticInitSample.java"
```

상수 풀의 14번째 항목에 `#14 = Utf8               <clinit>`가 있고, 아래 코드 내용 `static {};` 아래 부분에 123으로 초기화 하는 부분이 있음을 확인할 수 있다.

다시 원래의 예제 코드인 SimpleClass로 돌아와보자. SimpleClass에는 정적 필드가 없으므로 초기화 과정에서 따로 수행되는 것은 없다. 초기화 과정까지 마쳤으면 이제 드디어 JVM에 의해 main 메서드가 호출될 차례다.


## main 메서드 호출

```java
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello, JVM
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 10: 0
        line 11: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
```

- getstatic
- ldc
- invokevirtual
- return


## 종료




