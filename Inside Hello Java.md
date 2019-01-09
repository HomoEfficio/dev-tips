# Inside Hello Java.md

Java 소스가 어떻게 실행되는지 알아보자.

# 컴파일

일반적으로 컴파일이라고 하면 어떤 언어로 된 소스 코드를 기계가 인식할 수 있는 네이티브 코드로 변환하는 과정을 의미하지만, 자바에서의 컴파일은 자바 언어로 된 코드를 JVM이 인식할 수 있는 JVM 명령어 코드(바이트코드)로 변환하는 것을 의미한다.  
드물지만 자바에서의 컴파일도 일반적인 의미의 컴파일처럼 기계가 인식할 수 있는 코드로 변환하는 과정을 의미할 때도 있다. 대표적으로 `JIT 컴파일러`가 하는 컴파일은 바이트코드로 변환하는 것이 아니라 바이트코드로 변환된 것을 네이티브 코드로 변환하는 것을 의미한다.

## 컴파일의 단계

자바 소스 코드를 컴파일하는 과정이 몇 단계로 구성되는지 구체적으로 스펙에 명시되어 있지는 않다. 참고로 C 프로그램을 컴파일하는 과정은 보통 다음과 같이 4 단계로 구분한다.

1. Pre-processing
1. Compiling
1. Assembly
1. Linking

참고: https://www.geeksforgeeks.org/compiling-a-c-program-behind-the-scenes/

# 실행 파일 생성

## Preprocess


## Compile
하지만 실행의 대비되는 개념, 또는 실행 파일을 만들어 내는 모든 과정을 통틀어 컴파일이라고 표현하기도 한다.

1. Lexical Analysis: 토큰 스트림(Token Stream) 생성
예시 그림
1. Syntax Analysis: 추상 구문 트리(Abstract Syntax Tree) 생성 - 문법 체크
예시 그림
1. Symantic Analysis: 파스 트리 생성(Parse Tree) - 타입 체크 수행
예시 그림
1. Intermediate Code Generation: 중간 코드 생성 - 자바의 바이트코드가 중간 코드에 해당
런타임 상수 풀이 만들어짐. 런타임 상수 풀 안에는 가상 메모리 상에서의 주소가 아니라 심볼릭 참조가 들어있음
예시 그림
1. Code Optimization: 최적화 된 중간 코드 생성 - 코드 인라인화, 메모리 최적화 등
예시 그림

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

