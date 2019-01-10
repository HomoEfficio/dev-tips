# Inside Hello Java.md

Hello Java 소스 코드가 어떻게 컴파일 되고 실행되는지 살짝 깊게 알아보자.

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

자바는 컴파일 결과로 나온 바이트코드가 JVM에 의해 실행되면서 기계어 코드로 변환되므로, 실행 전에 기계어 코드를 만들어내는 어셈블리 단계가 없다고 볼 수 있다. 마찬가지로 링크 단계도 실행 전에 수행되지 않고 JVM에 의해 실행되면서 동적으로 수행된다.

컴파일과 어셈블리 과정을 하나로 뭉쳐서 컴파일이라고 하기도 한다. 아래에서 살펴볼 컴파일 세부 단계는 컴파일과 어셈블리 과정을 하나로 뭉친 개념이다.


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

### 3. Symantic Analysis(의미 분석)

의미 분석 단계에서는 타입 검사, 자동 타입 변환 등이 수행된다. 예를 들어 다음과 같은 코드는 구문 분석 단계에서는 에러가 나지 않지만, 의미 분석 단계에서는 타입 검사가 수행되면서 에러가 발생한다.

```
int a = "Hello";
```

### 4. Intermediate Code Generation(중간 코드 생성)

의미 분석 단계를 통과한 파스 트리를 바탕으로 기계어로 변환하기 좋은 형태의 중간 언어로 된 중간 코드를 생성한다. 중간 코드를 만들어 사용하는 이유는 중간 언어가 없을 때의 문제점을 그림으로 보면 금방 이해가 된다.

![Imgur](https://i.imgur.com/wROkyp1.jpg)

(그림 출처: https://www.slideshare.net/RamchandraRegmi/intermediate-code-generationramchandra-regmi)

한 마디로 간접화를 통해 경우의 수를 낮추고 효율을 높이기 위해 중간 코드를 생성한다.

자바의 바이트코드가 바로 이 중간 코드에 해당한다고 볼 수 있다. 위 그림에서 4개의 언어를 나타내는 네모를 각각 자바, 클로저(Clojuer), 스칼라, 코틀린이라고 하고, 녹색 네모를 바이트코드라고 생각하면 쉽게 이해할 수 있다.

어휘 분석에서 만들어져서, 구문 분석, 의미 분석 과정을 거치며 다듬어진 심볼 테이블은 중간 코드 생성 단계에서 클래스나 인터페이스별 [상수 풀(Constant Pool)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.4)을 만드는 데 사용된다. 

상수 풀 안에 담겨 있는 대부분의 자료구조는 정적인 엔티티를 나타내지만, `CONSTANT_Dynamic_info`, `CONSTANT_InvokeDynamic_info`로 표현되는 자료구조는 런타임에 정해지는 동적인 엔티티를 나타낸다. 

상수 풀에 저장된 정보는 해당 클래스나 인터페이스가 실제 생성될 때 [런타임 상수 풀(Run-Time Constant Pool)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.1)을 구성하는데 사용된다.

### 5. Code Optimization(중간 코드 최적화)

말 그대로 중간 코드가 더 효율적인 기계어로 변환되도록 최적화하는 과정이 수행된다. 

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

이 외에도 다양한 최적화 기법이 사용된다.

(참고: 컴파일러의 이해 - http://www.hanbit.co.kr/store/books/look.php?p_code=B4565472056)


## 링크(Link)

자바는 바이트코드를 실행할 때 링크 과정이 수행된다. 런타임 상수 풀 안에 있는 심볼릭 참조가 가상메모리 상에서의 주소로 대체됨

다른 바이트코드(라이브러리 등)를 연결해서 최종 네이티브 코드 생성


# 실행

운영체제가 JVM 프로세스를 실행

실행된 JVM 프로세스는 부트스트랩 클래스로더를 실행

부트스트랩 클래스로더는 Platform Classloader를 로딩

Platform Classloader는 System Clasloader를 로딩

System Classloader는 `public static void main(String[])`이 포함된 클래스 A를 로딩

A 클래스를 로딩, 확인, 초기화



 `public static void main(String[])`를 포함하고 있는 





컴파일 과정은 


자바 컴파일러에 의해 자바 코드가 어떤 바이트코드로 변환되는지는 [JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-3.html)에 예제와 함께 잘 나와 있다.



# 실행

