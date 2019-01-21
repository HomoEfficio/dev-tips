# Back to the Essence - Java 컴파일에서 실행까지 - (1)

Java 소스 코드가 어떻게 컴파일되고 실행되는지 살짝 깊게 알아보자.

# 컴파일

전통적으로 컴파일이라고 하면 어떤 언어로 된 소스 코드를 기계가 인식할 수 있는 네이티브 코드로 변환하는 과정을 의미하지만, 자바에서의 컴파일은 자바 언어로 된 코드를 JVM이 인식할 수 있는 JVM 명령어 코드(바이트코드)로 변환하는 것을 의미한다.  
드물지만 자바에서의 컴파일도 일반적인 의미의 컴파일처럼 기계가 인식할 수 있는 코드로 변환하는 과정을 의미할 때도 있다. 대표적으로 JIT 컴파일러가 하는 컴파일은 바이트코드로 변환하는 것이 아니라 바이트코드를 네이티브 코드로 변환하는 것을 의미한다.

## 실행 파일 생성 과정

자바 소스 코드를 컴파일하는 과정이 몇 단계로 구성되는지 구체적으로 스펙에 규정되어 있지는 않다. 참고로 C로 작성된 코드로부터 실행 파일을 만드는 과정은 보통 다음과 같이 4 단계로 구분한다.

1. 전처리(Pre-processing)
    - 주석 제거
    - 매크로 인라인화
    - include 파일 인라인화
1. 컴파일(Compiling)
    - 컴파일러가 전처리 과정을 거친 C 소스 코드를 컴파일해서 어셈블리어 코드로 변환
1. 어셈블리(Assembly)
    - 어셈블러가 어셈블리어 코드를 기계어 코드로 변환
1. 링크(Linking)
    - 링커가 기계어 코드와 공유 라이브러리 등 다른 코드를 합쳐서 최종 실행 파일 생성

(참고: https://www.geeksforgeeks.org/compiling-a-c-program-behind-the-scenes/)

컴파일과 어셈블리 단계를 그냥 컴파일 단계 하나로 합쳐서 보면 다음과 같은 그림으로 쉽게 이해할 수 있다.

1. 전처리

    ![Imgur](https://i.imgur.com/6Gixn4m.png)

1. 컴파일

    ![Imgur](https://i.imgur.com/XgKJv8x.png)

1. 링크

    ![Imgur](https://i.imgur.com/bjruQVw.png)

(그림 참고: https://www.sitesbay.com/cprogramming/c-compile-link-program)

참고로 중요하진 않지만 자바는 전처리 과정에서 주석이 있던 행 자체가 제거되지는 않는다. 바이트코드 내용 중에 자바 소스 코드의 행 번호와 바이트코드 명령어의 위치를 매핑해주는 부분이 있는데 이 때 표시되는 자바 소스 코드 행 번호는 주석이 있던 행이 제거되지 않은 상태 기준의 행 번호가 표시된다.

자바는 컴파일 결과로 나온 바이트코드가 JVM에 의해 실행되면서 네이티브 기계어 코드로 변환되므로, 프로그램 실행 전에 네이티브 기계어 코드를 만들어내는 어셈블리 단계가 없다고 볼 수 있다. 마찬가지로 링크 단계도 프로그램 실행 전에 수행되지 않고 JVM에 의해 프로그램이 실행될 때 동적으로 수행된다.

따라서 자바의 컴파일 절차는 아주 단순하다. 그림조차도 그릴 필요 없고 다음과 같이 표현할 수 있다.

```
자바 소스 코드 파일(.java) -> javac 컴파일러 -> JVM 바이트코드(.class)
```

앞에서 나온 C 컴파일 과정 그림에서 살펴본 것처럼 컴파일과 어셈블리 과정을 하나로 뭉쳐서 컴파일이라고 하기도 한다. 아래에서 살펴볼 일반적인 컴파일 세부 단계는 컴파일과 어셈블리 과정을 하나로 뭉친 개념이다.


## 컴파일 세부 단계

### 1. Lexical Analysis(어휘 분석)

Lexical Analyzer(Lexer 또는 Tokenizer라고도 한다)가 소스 코드에서 문자 단위로 읽어서 어휘소(lexeme)를 식별하고 어휘소를 설명하는 토큰 스트림(Token Stream)을 생성한다.

어휘소는 식별가능한 문자 시퀀스인데 다음과 같은 것들을 통칭한다.

- 키워드(keywords): `public`, `class`, `main`, `for` 등
- 리터럴(literals): `1L`, `2.3f`, `"Hello"` 등
- 식별자(identifiers): 변수 이름, 상수 이름, 함수 이름 등
- 연산자(operators): `+`, `-` 등
- 구분 문자(punctuation characters): `,`, `[]`, `{}`, `()` 등

토큰(Token)은 타입(키워드, 리터럴, 식별자 등)과 값(`public`, `1L`, `main` 등)으로 구성되며 어휘소를 설명하는 객체로 볼 수 있다.

식별자 토큰은 어휘 분석 단계에서 심볼 테이블에 저장되고 이후 단계에서 계속 사용된다.

### 2. Syntax Analysis(구문 분석)

Syntax Analyzer(구문 분석기, 파서(Parser)라고도 한다)가 어휘 분석 결과로 나온 토큰 스트림이 언어의 스펙으로 정해진 문법 형식에 맞는지 검사해서, 맞지 않으면 컴파일 에러를 내고, 맞으면 파스 트리(Parse Tree)를 생성한다(구문 분석 단계의 결과로 나오는 파스 트리를 추상 구문 트리(Abstract Syntax Tree)라고 부르는 자료도 있다).

어휘 분석과 구문 분석 과정을 그림으로 요약하면 다음과 같다.

![Imgur](https://i.imgur.com/ynDWnl2.gif)

(그림 출처: https://en.wikipedia.org/wiki/Compiler)

위 그림에서 Parser 아래에 있는 트리가 파스 트리다.

### 3. Symantic Analysis(의미 분석)

의미 분석 단계에서는 타입 검사, 자동 타입 변환 등이 수행된다. 예를 들어 다음과 같은 코드는 구문 분석 단계에서는 에러가 나지 않지만, 의미 분석 단계에서는 타입 검사가 수행되면서 에러가 발생한다.

```
int a = "Hello";
```
의미 분석 단계를 거치면서 파스 트리에 타입 관련 정보 등이 추가된다.

### 4. Intermediate Code Generation(중간 코드 생성)

의미 분석 단계를 통과한 파스 트리를 바탕으로 기계어로 변환하기 좋은 형태의 중간 언어로 된 중간 코드를 생성한다. 중간 코드를 만들어 사용하는 이유는 중간 언어가 없을 때의 문제점을 그림으로 보면 금방 이해가 된다.

![Imgur](https://i.imgur.com/wROkyp1.jpg)

(그림 출처: https://www.slideshare.net/RamchandraRegmi/intermediate-code-generationramchandra-regmi)

한 마디로 중간 단계를 하나 둬서 간접화를 통해 경우의 수를 낮추고 효율을 높이기 위해 중간 코드를 생성한다.

자바의 바이트코드가 바로 이 중간 코드에 해당한다고 볼 수 있다. 위 그림에서 4개의 언어를 나타내는 네모를 각각 자바, 클로저(Clojure), 스칼라, 코틀린이라고 하고, 녹색 네모를 바이트코드라고 생각하면 쉽게 이해할 수 있다.

어휘 분석에서 만들어져서, 구문 분석, 의미 분석 과정을 거치며 다듬어진 심볼 테이블은 중간 코드인 바이트코드 생성 단계에서 클래스나 인터페이스별 [상수 풀(Constant Pool)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.4)을 만드는 데 사용된다. 

상수 풀 안에 담겨 있는 대부분의 자료구조는 이름, 설명자(descriptor), 값 등 테이블에 정적으로 저장된 정보를 조합해서 엔티티를 직접 표현하지만, `CONSTANT_Dynamic_info`, `CONSTANT_InvokeDynamic_info`로 표현되는 자료구조는 런타임에 정해지는 동적인 엔티티를 간접적으로 표현한다.(참고: https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.4.10)

상수 풀에 저장된 정보는 해당 클래스나 인터페이스가 실제 생성될 때 [런타임 상수 풀(Run-Time Constant Pool)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.1)을 구성하는데 사용된다.

### 5. Code Optimization(중간 코드 최적화)

말 그대로 중간 코드가 더 효율적인 기계어로 변환되도록 최적화하는 과정이 수행된다. 다음과 같이 매우 다양한 최적화 기법이 사용된다.

#### 핍홀(Peephole) 최적화

- 중복 명령어 제거
- 도달 불가능한 코드 제거
- 제어 흐름 최적화
- 비용 낮은 연산자로 변환 등

#### 지역 최적화

- 지역 공통 부분식 제거
- 복사 전파
- 상수 폴딩 등

#### 루프 최적화

- 코드 이동
- 귀납 변수 최적화
- 루프 융합/교환/전개 등

#### 전역 최적화

- 전역 공통 부분식 제거
- 상수 폴딩 등

이 외에도 다양한 최적화 기법이 사용되는데, 쉽게 감이 오는 루프 최적화의 코드 이동만 확인해보자. 실제로 최적화되는 것은 바이트코드지만 보기 편하게 자바 코드로 표현한다.

```java
for (int i = 0 ; i < 100000 ; i++) {
    c[k] = 2 * (p - q) * (n - k + 1) / (sqrt(n) + n);
}

// i와 관계 없이 값이 고정되어 있는 식을 반복문 밖으로 옮겨서 불필요한 계산 반복을 제거

factor = 2 * (p - q);
denominator = (sqrt(n) + n);
for (int i = 0 ; i < 100000 ; i++) {
    c[k] = factor * (n - k + 1) / denominator;
}
```

(참고: 컴파일러의 이해 - http://www.hanbit.co.kr/store/books/look.php?p_code=B4565472056)

## 컴파일 과정 정리

자바의 컴파일 과정은 여기까지다. 자바의 컴파일 과정을 한 마디로 요약하면 **자바 코드를 자바 언어 스펙에 따라 분석/검증하고, JVM 스펙의 class 파일 구조에 맞는 바이트코드를 만들어내는 과정** 이라고 할 수 있다.

바이트코드는 로딩, 링크 과정을 거쳐야 하지만 분명히 JVM에서 실행될 수 있는 코드다. 따라서 꼭 자바 언어 스펙을 따르는 자바가 아니라도, JVM 스펙의 class 파일 구조에 맞는 바이트코드를 만들어 낼 수 있다면 어떤 언어든 JVM에서 실행될 수 있다. 클로저(Clojure)나 스칼라, 코틀린 등이 JVM에서 실행될 수 있는 이유가 바로 여기에 있다.

자바 코드의 변수, 상수, 제어문, 연산, 인자, 메서드 호출, 배열, switch문, 예외 처리, finally, synchronization, 애너테이션, 모듈(Java 9 이후) 등이 바이트코드로 어떻게 변환되는지는 [JVM 스펙의 3장](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-3.html)에 나오는 예시를 통해 확인할 수 있다.

그냥 지나치면 허전하니 간단한 자바 파일과 컴파일 된 바이트코드를 한 번 살펴보자.

## 바이트코드 구경하기

그냥 헬로월드는 너무 단순하니까 인터페이스를 사용하는 코드 예제를 살펴보자. main 메서드를 가진 `GreetingMain` 클래스가 `Greeting` 인터페이스를 구현하는 `KoreanGreeting` 클래스를 사용하는 예제다.

먼저 인터페이스인 `Greeting`부터 살펴보자.

### Greeting

```java
package homo.efficio.jvm.sample;

public interface Greeting {

    String sayHello(String name);
}
```

메서드 하나를 가지고 있는 아주 단순한 인터페이스다. 컴파일 한 후에 다음과 같이 `javap` 명령으로 바이트코드를 확인할 수 있다. `javap`는 바이너리인 바이트코드 .class 파일을 텍스트로 보여주는 일종의 역어셈블러 프로그램이다.

>javap -v -l -p homo/efficio/jvm/sample/Greeting.class

```
Classfile /C:/gitrepo/scratchpad/plain-java-scratchpad/out/production/classes/homo/efficio/jvm/sample/Greeting.class
  Last modified 2019. 1. 13.; size 181 bytes
  MD5 checksum 8f7ae541e0a64f511d820930f739d4ac
  Compiled from "Greeting.java"
public interface homo.efficio.jvm.sample.Greeting
  minor version: 0
  major version: 53
  flags: (0x0601) ACC_PUBLIC, ACC_INTERFACE, ACC_ABSTRACT
  this_class: #1                          // homo/efficio/jvm/sample/Greeting
  super_class: #2                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 1, attributes: 1
Constant pool:
  #1 = Class              #7              // homo/efficio/jvm/sample/Greeting
  #2 = Class              #8              // java/lang/Object
  #3 = Utf8               sayHello
  #4 = Utf8               (Ljava/lang/String;)Ljava/lang/String;
  #5 = Utf8               SourceFile
  #6 = Utf8               Greeting.java
  #7 = Utf8               homo/efficio/jvm/sample/Greeting
  #8 = Utf8               java/lang/Object
{
  public abstract java.lang.String sayHello(java.lang.String);
    descriptor: (Ljava/lang/String;)Ljava/lang/String;
    flags: (0x0401) ACC_PUBLIC, ACC_ABSTRACT
}
SourceFile: "Greeting.java"
```

인터페이스의 바이트코드는 Classfile, public interface ..., Constant pool, { 바이트코드 }, SourceFile 이렇게 크게 5가지 항목으로 구분되어 표시된다.

컴파일과 실행 관점에서 주목해야할 항목은 상수 풀(Constant pool)과 실제 소스 코드로부터 변환된 바이트코드 내용이다. 

상수풀에는 `Class`와 `Utf8`로 분류되는 값들이 표시되어 있다. 상수 풀에 포함된 정보는 `#N`의 형식으로 인덱스되어 있다. `Class`는 말그대로 클래스임을 나타내고 `Utf8`은 클래스나 메서드 등의 이름을 나타내는 식별자를 UTF-8로 인코딩 된 값으로 나타내고 있다. `Class`로 분류된 항목의 값은 `#7` 같이 다른 항목을 가리키는 일종의 참조로 되어 있고, 참조를 통해 가리키는 항목의 값은 주석으로 병기(`// homo/efficio/jvm/sample/Greeting`)되어 있다.

바이트코드에는 원래 자바 소스에는 없던 `abstract`라는 키워드가 추가되어 표시되어 있다. sayHello 메서드의 파라미터 정보(`(Ljava/lang/String;)`) 와 반환 타입 정보(`Ljava/lang/String;`)가 descriptor 항목에 표시되고, 접근 지정자(`ACC_PUBLIC`, `ACC_ABSTRACT`)가 flags 항목에 표시된다. 

아주 간단해서 바이트코드의 상수풀과 바이트코드가 어떤 식으로 기술되는지 비교적 쉽게 감을 잡을 수 있다. 너무 간단해서 바이트코드 내용이 별로 없기 때문에, 바이트코드에 대한 설명은 구현 클래스인 `KoreanGreeting`에서 실제 코드와 함께 다시 살펴볼 것이다.

### KoreanGreeting

```java
package homo.efficio.jvm.sample;

public class KoreanGreeting implements Greeting {

    private String hello = "안녕 ";

    @Override
    public String sayHello(String name) {
        return getHello() + name;
    }

    private String getHello() {
        return this.hello;
    }
}
```

`Greeting` 인터페이스를 구현하고 있고, `hello`라는 필드를 하나 가지고 있는 단순한 클래스다. `getHello()`는 메서드가 2개일 때는 어떻게 표시되는지, 내부 private 메서드 호출은 어떻게 표시되는지 보기 위해 일부러 추가했다.

>javap -v -l -p homo/efficio/jvm/sample/KoreanGreeting.class

```
Classfile /C:/gitrepo/scratchpad/plain-java-scratchpad/out/production/classes/homo/efficio/jvm/sample/KoreanGreeting.class
  Last modified 2019. 1. 12.; size 1132 bytes
  MD5 checksum d7ac2a6fd38c67407480720ca730d987
  Compiled from "KoreanGreeting.java"
public class homo.efficio.jvm.sample.KoreanGreeting implements homo.efficio.jvm.sample.Greeting
  minor version: 0
  major version: 53
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #6                          // homo/efficio/jvm/sample/KoreanGreeting
  super_class: #7                         // java/lang/Object
  interfaces: 1, fields: 1, methods: 3, attributes: 3
Constant pool:
   #1 = Methodref          #7.#25         // java/lang/Object."<init>":()V
   #2 = String             #26            // ▒?▒
   #3 = Fieldref           #6.#27         // homo/efficio/jvm/sample/KoreanGreeting.hello:Ljava/lang/String;
   #4 = Methodref          #6.#28         // homo/efficio/jvm/sample/KoreanGreeting.getHello:()Ljava/lang/String;
   #5 = InvokeDynamic      #0:#32         // #0:makeConcatWithConstants:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
   #6 = Class              #33            // homo/efficio/jvm/sample/KoreanGreeting
   #7 = Class              #34            // java/lang/Object
   #8 = Class              #35            // homo/efficio/jvm/sample/Greeting
   #9 = Utf8               hello
  #10 = Utf8               Ljava/lang/String;
  #11 = Utf8               <init>
  #12 = Utf8               ()V
  #13 = Utf8               Code
  #14 = Utf8               LineNumberTable
  #15 = Utf8               LocalVariableTable
  #16 = Utf8               this
  #17 = Utf8               Lhomo/efficio/jvm/sample/KoreanGreeting;
  #18 = Utf8               sayHello
  #19 = Utf8               (Ljava/lang/String;)Ljava/lang/String;
  #20 = Utf8               name
  #21 = Utf8               getHello
  #22 = Utf8               ()Ljava/lang/String;
  #23 = Utf8               SourceFile
  #24 = Utf8               KoreanGreeting.java
  #25 = NameAndType        #11:#12        // "<init>":()V
  #26 = Utf8               ▒?▒
  #27 = NameAndType        #9:#10         // hello:Ljava/lang/String;
  #28 = NameAndType        #21:#22        // getHello:()Ljava/lang/String;
  #29 = Utf8               BootstrapMethods
  #30 = MethodHandle       6:#36          // REF_invokeStatic java/lang/invoke/StringConcatFactory.makeConcatWithConstants:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;
  #31 = String             #37            // \u0001\u0001
  #32 = NameAndType        #38:#39        // makeConcatWithConstants:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #33 = Utf8               homo/efficio/jvm/sample/KoreanGreeting
  #34 = Utf8               java/lang/Object
  #35 = Utf8               homo/efficio/jvm/sample/Greeting
  #36 = Methodref          #40.#41        // java/lang/invoke/StringConcatFactory.makeConcatWithConstants:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;
  #37 = Utf8               \u0001\u0001
  #38 = Utf8               makeConcatWithConstants
  #39 = Utf8               (Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #40 = Class              #42            // java/lang/invoke/StringConcatFactory
  #41 = NameAndType        #38:#46        // makeConcatWithConstants:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;
  #42 = Utf8               java/lang/invoke/StringConcatFactory
  #43 = Class              #48            // java/lang/invoke/MethodHandles$Lookup
  #44 = Utf8               Lookup
  #45 = Utf8               InnerClasses
  #46 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;
  #47 = Class              #49            // java/lang/invoke/MethodHandles
  #48 = Utf8               java/lang/invoke/MethodHandles$Lookup
  #49 = Utf8               java/lang/invoke/MethodHandles
{
  private java.lang.String hello;
    descriptor: Ljava/lang/String;
    flags: (0x0002) ACC_PRIVATE

  public homo.efficio.jvm.sample.KoreanGreeting();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc           #2                  // String ▒?▒
         7: putfield      #3                  // Field hello:Ljava/lang/String;
        10: return
      LineNumberTable:
        line 3: 0
        line 5: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lhomo/efficio/jvm/sample/KoreanGreeting;

  public java.lang.String sayHello(java.lang.String);
    descriptor: (Ljava/lang/String;)Ljava/lang/String;
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: invokespecial #4                  // Method getHello:()Ljava/lang/String;
         4: aload_1
         5: invokedynamic #5,  0              // InvokeDynamic #0:makeConcatWithConstants:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
        10: areturn
      LineNumberTable:
        line 9: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lhomo/efficio/jvm/sample/KoreanGreeting;
            0      11     1  name   Ljava/lang/String;

  private java.lang.String getHello();
    descriptor: ()Ljava/lang/String;
    flags: (0x0002) ACC_PRIVATE
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #3                  // Field hello:Ljava/lang/String;
         4: areturn
      LineNumberTable:
        line 13: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lhomo/efficio/jvm/sample/KoreanGreeting;
}
SourceFile: "KoreanGreeting.java"
InnerClasses:
  public static final #44= #43 of #47;    // Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #30 REF_invokeStatic java/lang/invoke/StringConcatFactory.makeConcatWithConstants:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #31 \u0001\u0001
```

자바 소스 코드 상으로는 `Greeting` 인터페이스와 몇 줄 차이 안 나는데 바이트코드의 양은 큰 차이가 난다. 바이트코드를 분석하는 것이 글의 목적이 아니라 컴파일이라는 큰 과정을 살펴보면서 결과물인 바이트코드도 눈으로 구경해보자는 취지이므로 개략적인 생김새와 기본적인 내용만 훑어보자.

#### 상수 풀

상수 풀에는 `Methodref`, `String`, `Fieldref`, `Methodref`, `InvokeDynamic`, `NameAndType`, `MethodHandle` 등 새로운 종류의 상수 항목이 나오는데, 이름과 값을 조금 살펴보면 어떻게 사용되는지 대략 감을 잡을 수 있다. 소스 코드 수준에서 정적으로 파악할 수 있는 변수, 상수, 메서드 등의 일람표라고 생각하면 된다.

상수 풀에 저장되는 상수 항목의 종류는 총 17개이며, 자세한 내용은 [JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.4)을 참고한다.


#### 바이트코드

{} 로 묶여서 표시되는 바이트코드는 대략 다음과 같은 구조로 되어 있다.

- 필드나 메서드 선언부
    - descriptor: 필드의 타입이나 메서드의 파라미터 및 반환 타입
    - flags: 접근 지정자
    - Code
        - stack, locals, args_size: 스택 높이, 로컬 변수 갯수, 인자 갯수
            - 실제 구현 코드: 코드 위치, 바이트코드 명령어(instruction), 오퍼랜드(operand, 피연산자)
        - LineNumberTable: 자바 코드의 행 번호와 바이트코드의 위치 매핑 테이블
        - LocalVariableTable: 로컬 변수 테이블

어셈블리어 프로그래밍 경험이 있는 개발자에게는 바이트코드가 그리 낯설지 않을 것이다. 바이트코드의 대부분은 오퍼랜드 스택에 값을 넣고, 빼고, 읽고, 복사하고, 스왑하거나 메서드를 호출하는 내용을 담고 있다.

바이트코드 명령어에 대한 자세한 내용은 [JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5)을 참고하고 여기에서는 메서드 호출과 관계있는 `invoke*` 명령어만 짧게 알아보자.

명령어 이름 | 하는 일
--- | ---
invokeinterface | 인터페이스에 정의된 메서드 호출
invokespecial | 생성자, 수퍼클래스의 메서드, 현재 클래스의 메서드 등 객체 참조(`obj.`) 없이 메서드 이름만으로 호출되는 메서드 호출
invokestatic | 정적 메서드 호출
invokevirtual | 자바 메서드 호출의 기본 방식이며, 객체 참조(`obj.`)를 붙여서 호출되는 일반적인 메서드 호출
invokedynamic | JVM에서 실행되는 동적 타입 언어를 위해 Java 7에 추가된 명령어. 람다식도 invokedynamic을 이용해서 구현되었다. 자세한 내용은 [오라클 문서](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/multiple-language-support.html)나 [네이버 문서](https://d2.naver.com/helloworld/4911107) 또는 [DZone 문서](https://dzone.com/articles/dismantling-invokedynamic)를 참고하자.

한 가지 눈여겨 볼 것은 실제 자바 소스 코드에는 없던 디폴트 생성자가 추가되어 있다는 점이다. 컴파일러가 자동으로 추가해준다는 사실을 실제로 확인한 셈이다. 디폴트 생성자는 [자바 언어 스펙](https://docs.oracle.com/javase/specs/jls/se11/html/jls-8.html#jls-8.8.9)을 참고하자.

### GreetingMain

`Greeting` 인터페이스와 이를 구현한 `KoreanGreeting` 클래스를 사용해서 인사말을 찍는 클래스다.

```java
package homo.efficio.jvm.sample;

import java.lang.reflect.InvocationTargetException;

public class GreetingMain {

    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {

        KoreanGreeting koreanGreeting = new KoreanGreeting();
        System.out.println(koreanGreeting.sayHello("Homo Efficio"));

        Greeting greeting = new KoreanGreeting();
        System.out.println(greeting.sayHello("Homo Efficio"));

        sayHelloFromDynamicallyLoadedClass(args[0]);
    }

    private static void sayHelloFromDynamicallyLoadedClass(String arg) throws ClassNotFoundException, InstantiationException, IllegalAccessException, InvocationTargetException, NoSuchMethodException {
        ClassLoader classLoader = GreetingMain.class.getClassLoader();
        Class<?> aClass = classLoader.loadClass(arg);
        if (Greeting.class.isAssignableFrom(aClass)) {
            Greeting aGreeting = (Greeting) aClass.getDeclaredConstructor().newInstance();
            System.out.println(aGreeting.sayHello("Homo Efficio"));
        }
    }
}
```

바이트코드 대략적인 구조 설명은 앞에서 했으므로 여기에서는 인터페이스를 통한 자바의 다형성이 발현되는 지점을 알 수 있는 부분만 살펴보자. 나머지 내용이 궁금하다면 https://github.com/HomoEfficio/plain-java-scratchpad/tree/master/src/main/java/homo/efficio/jvm/sample 를 참고한다.

```
Constant pool:
    ...
    #8 = InterfaceMethodref #16.#65       // homo/efficio/jvm/sample/Greeting.sayHello:(Ljava/lang/String;)Ljava/lang/String;
    ...
{
  public static void main(java.lang.String[]) throws java.lang.ClassNotFoundException, java.lang.NoSuchMethodException, java.lang.IllegalAccessException, java.lang.reflect.InvocationTargetException, java.lang.InstantiationException;
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=4, args_size=1
         0: new           #2                  // class homo/efficio/jvm/sample/KoreanGreeting
         3: dup
         4: invokespecial #3                  // Method homo/efficio/jvm/sample/KoreanGreeting."<init>":()V
         7: astore_1
         8: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        11: aload_1
        12: ldc           #5                  // String Homo Efficio
        14: invokevirtual #6                  // Method homo/efficio/jvm/sample/KoreanGreeting.sayHello:(Ljava/lang/String;)Ljava/lang/String;
        17: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        20: new           #2                  // class homo/efficio/jvm/sample/KoreanGreeting
        23: dup
        24: invokespecial #3                  // Method homo/efficio/jvm/sample/KoreanGreeting."<init>":()V
        27: astore_2
        28: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        31: aload_2
        32: ldc           #5                  // String Homo Efficio
        34: invokeinterface #8,  2            // InterfaceMethod homo/efficio/jvm/sample/Greeting.sayHello:(Ljava/lang/String;)Ljava/lang/String;
        39: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        ...
}
```

상수 풀의 8번쨰 항목에 `InterfaceMethodref`라는 항목으로 `Greeting` 인터페이스의 sayHello 메서드가 등록되어 있다.

자바 소스코드에서 아래와 같이 인터페이스를 사용하지 않는 부분은

```java
        KoreanGreeting koreanGreeting = new KoreanGreeting();
        System.out.println(koreanGreeting.sayHello("Homo Efficio"));
```

다음과 같이 `invokevirtual`이 사용되고,

```
        12: ldc           #5                  // String Homo Efficio
        14: invokevirtual #6                  // Method homo/efficio/jvm/sample/KoreanGreeting.sayHello:(Ljava/lang/String;)Ljava/lang/String;
        17: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
```

인터페이스를 사용하는 아래 코드는

```java
        Greeting greeting = new KoreanGreeting();
        System.out.println(greeting.sayHello("Homo Efficio"));
```

다음과 같이 `invokeinterface`가 사용됨을 확인할 수 있다.

```
        32: ldc           #5                  // String Homo Efficio
        34: invokeinterface #8,  2            // InterfaceMethod homo/efficio/jvm/sample/Greeting.sayHello:(Ljava/lang/String;)Ljava/lang/String;
        39: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
```

바이트코드 구경은 여기서 줄인다. 더 궁금하다면 알고 싶은 부분을 직접 코딩/컴파일하고 `javap`와 JVM 스펙으로 확인해보는 것이 가장 좋다.


# 1탄 마무리

여기까지 자바 소스 코드가 바이트코드로 어떻게 컴파일되는지 알아봤다. 짧게 정리해보면 다음과 같다.

>자바도 전처리, 컴파일, 링크 과정을 통해 최종 실행 파일이 만들어진다.
>
>컴파일의 세부 단계는 어휘 분석, 구문 분석, 의미 분석, 중간 코드 생성, 중간 코드 최적화로 구성된다.  
>
>**자바 컴파일은 자바 코드를 자바 언어 스펙에 따라 분석/검증하고, JVM 스펙의 class 파일 구조에 맞는 바이트코드를 만들어내는 과정**이다.
>
>자바 소스 코드를 컴파일한 결과로 나오는 class 파일은 크게 보면 **클래스 메타 정보, 상수 풀, 코드 구현부(JVM 명령어+오퍼랜드)로 구성**된다.
>
>**소스 코드에서 정적으로 파악할 수 있는 변수, 상수, 메서드 등의 정보가 클래스 파일 단위의 상수 풀(Constant Pool)에 저장**되고,  
>**코드 구현부는 상수 풀에 저장된 항목을 오퍼랜드로 사용해서 실행되는 JVM 명령어로 변환**된다.
>
>javap 명령으로 바이너리 바이트코드를 눈으로 읽을 수 있는 텍스트로 역어셈블해서 확인할 수 있다.


