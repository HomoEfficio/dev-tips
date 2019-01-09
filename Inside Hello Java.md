# Inside Hello Java.md

Hello Java 소스 코드가 어떻게 컴파일 되고 실행되는지 살짝 깊게 알아보자.

# 컴파일

전통적으로 컴파일이라고 하면 어떤 언어로 된 소스 코드를 기계가 인식할 수 있는 네이티브 코드로 변환하는 과정을 의미하지만, 자바에서의 컴파일은 자바 언어로 된 코드를 JVM이 인식할 수 있는 JVM 명령어 코드(바이트코드)로 변환하는 것을 의미한다.  
드물지만 자바에서의 컴파일도 일반적인 의미의 컴파일처럼 기계가 인식할 수 있는 코드로 변환하는 과정을 의미할 때도 있다. 대표적으로 `JIT 컴파일러`가 하는 컴파일은 바이트코드로 변환하는 것이 아니라 바이트코드를 네이티브 코드로 변환하는 것을 의미한다.

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

참고: https://www.geeksforgeeks.org/compiling-a-c-program-behind-the-scenes/

자바는 컴파일 결과로 나온 바이트코드가 JVM에 의해 실행되면서 기계어 코드로 변환되므로, 실행 전에 기계어 코드를 만들어내는 어셈블리 단계가 없다고 볼 수 있다. 마찬가지로 링크 단계도 실행 전에 수행되지 않고 JVM에 의해 실행되면서 동적으로 수행된다.

컴파일과 어셈블리 과정을 하나로 뭉쳐서 컴파일이라고 하기도 한다. 아래에서 살펴볼 컴파일 세부 과정은 컴파일과 어셈블리 과정을 하나로 뭉친 개념이다.


## 컴파일 세부 과정

### 1. Lexical Analysis(어휘 분석)

Lexical Analyzer(Lexer 또는 Tokenizer라고도 한다)가 소스 코드에서 문자 단위로 읽어서 어휘소(lexeme)를 식별하고 어휘소를 설명하는 토큰 스트림(Token Stream)을 생성한다.

어휘소는 식별가능한 문자 시퀀스인데 다음과 같은 것들을 통칭한다.

- 키워드(keywords): `public`, `class`, `main`, `for` 등
- 리터럴(literals): `1L`, `2.3f`, `"Hello"` 등
- 식별자(identifiers): 변수 이름, 상수 이름, 함수 이름 등
- 연산자(operators): `+`, `-` 등
- 구분 문자(punctuation characters): `,`, `[]`, `{}`, `()` 등

토큰(Token)은 타입(키워드, 리터럴, 식별자 등)과 값(`public`, `1L`, `main` 등)으로 구성되며 어휘소를 설명하는 객체로 볼 수 있다.

### 2. Syntax Analysis(문법 분석)

추상 구문 트리(Abstract Syntax Tree) 생성 - 문법 체크

예시 그림

### 3. Symantic Analysis: 파스 트리 생성(Parse Tree) - 타입 체크 수행
예시 그림

### 4. Intermediate Code Generation: 중간 코드 생성 - 자바의 바이트코드가 중간 코드에 해당

런타임 상수 풀이 만들어짐. 런타임 상수 풀 안에는 가상 메모리 상에서의 주소가 아니라 심볼릭 참조가 들어있음
예시 그림

### 5. Code Optimization: 최적화 된 중간 코드 생성 - 코드 인라인화, 메모리 최적화 등
예시 그림

참고: https://www.kttpro.com/2017/02/09/six-phases-of-the-compilation-process/

## Link

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

