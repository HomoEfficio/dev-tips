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

`java` 명령어의 인자로 지정한 설정 옵션에 맞게 JVM이 실행되고, JVM이 클래스로더를 이용해서 initial class를 create하고, initial class를 link하고, initialize하고, main 메서드를 호출한다.([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.2) 참고)

![]()  // JRE 시작 그림


### 용어 정리

몇 가지 용어를 일부러 스펙에 나온 원어 그대로 썼는데 의미는 다음과 같다.

- initial class: JVM 구현에 따라 다를 수 있지만 일반적으로 main 메서드를 포함하는 클래스로서 java 명령어의 인자로 지정되는 클래스 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.2) 참고)
- create (a class or interface): 해당 클래스나 인터페이스의 바이트코드를 로딩해서 JVM이 할당한 메모리(Method Area, 메서드 영역)에 construction하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.3) 참고)
- link (a class or interface): 해당 클래스나 인터페이스의 바로 위 수퍼클래스나 수퍼인터페이스, 또는 배열일 경우 배열의 원소인 클래스나 인터페이스를 확인(verify)/준비(prepare)하고, 심볼릭 참조를 해석(resolve)하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4) 참고)
- initialize (a class or interface): 해당 클래스나 인터페이스의 initialization method를 실행하는 것 ([JVM 스펙](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.5) 참고)

앞으로 initial class는 시작 클래스, create은 생성이라고 하면 힙에 객체를 생성하는 것과 혼동될 수 있으므로 그냥 create라고 쓰고, link는 링크, initialize는 초기화라고 쓴다.


## 런타임 데이터 영역

`java` 명령어로 자바 애플리케이션을 실행하면 JVM이 실행되면서 시작 클래스를 create, 링크, 초기화하고 main 메서드를 호출한다고 했다. 시작 클래스를 create 한다는 것은 시작 클래스의 바이트코드를 읽어서 JVM의 메모리 어딘가에 쓰는 것을 의미한다. JVM의 메모리는 어떻게 생겼을까?

JVM은 프로그램의 실행에 사용되는 메모리를 런타임 데이터 영역(Runtime Data Area)이라고 부르는 몇 가지 영역으로 나눠서 관리한다. 스펙의 목차로 보면 밋밋하게 다음과 같이 나열되어 있다.

1. PC 레지스터
1. JVM 스택
1. 힙(Heap)
1. 메서드 영역(Method Area)
1. 런타임 상수 풀(Run-Time Constnat Pool)
1. 네이티브 메서드 스택

이렇게 보면 위 6가지가 동등한 최상위 수준에서 분류되는 것처럼 보인다. 그래서 검색해보면 다음과 같이 표현된 그림을 보게 되는데, 실제 스펙의 설명을 보면 다음과 같이 구분하는 것이 더 적합하다.

1. JVM 단위
    1. 힙
    1. 메서드 영역

1. 클래스(인터페이스) 단위
    1. 런타임 상수 풀

1. 스레드 단위
    1. PC 레지스터
    1. JVM 스택
    1. 네이티브 메서드 스택

