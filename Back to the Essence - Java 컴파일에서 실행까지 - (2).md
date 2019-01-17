# Back to the Essence - Java 컴파일에서 실행까지 - (2)

Java 소스 코드가 어떻게 컴파일되고 실행되는지 살짝 깊게 알아보자.

# 실행

자바 애플리케이션은 `java` 명령어로 실행할 수 있다. [오라클의 Tools Reference 문서](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE)에 나오는 `java`에 대한 설명은 다음과 같다.

>`java` 명령어는 자바 애플리케이션을 시작한다.  
>`java` 명령어는 먼저 JRE(Java Runtime Environment)를 시작하고,  
>인자로 지정된 클래스(`public static void main(String[] args)`를 포함하고 있는 클래스)를 로딩하고,  
>`main()` 메서드를 호출한다.


## JRE vs JVM

`java`는 JRE를 시작한다고 하니, JRE와 JVM의 관계에 대해 간결하게 짚고 넘어가자. JRE는 Java Runtime Environment, 말 그대로 자바 실행 환경이다. 그래서 JRE는  바이트코드(class 파일)를 실행하는 JVM을 포함하고 있고, 그 외에 바이트코드 실행 시 사용되는 여러 util, math 등 자바 패키지를 포함한다.

따라서 다음과 같이 간단하게 요약할 수 있다.

>JRE = JVM + 여러 지원 패키지


## JRE 시작

// java 의 main 메서드 찾아서 내용 추가


## 메모리 구조 그림


## 클래스로딩

// native bootstrap 클래스로더를 실행하고, bootstrap 클래스로더가 rt.jar 등을 실행하고, 플랫폼/앱 클래스로더가 로딩하는 내용


## 링크

// verification, preparation, resolution


## main 메서드 호출

// jvm 스펙의 5.5 Initialization 참고
// 필요한 클래스로딩 및 실행 과정
