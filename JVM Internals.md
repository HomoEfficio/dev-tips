# JVM Internals

> http://blog.jamesdbloom.com/JVMInternals.html 를 요약 정리한 글

Java Virtual Machine(JVM)의 내부 아키텍처를 알아보자. 아래의 다이어그램은 Java 7 SE를 준수하는 전형적인 JVM의 주요 내부 구성요소를 보여주고 있다.

![Imgur](https://i.imgur.com/DeLfwDA.png)

그림에 나오는 구성요소는 두 개의 단원으로 나누어 설명한다. 첫 번째 단원에서는 각 스레드별로 생성되는 구성요소를 다루고, 두 번째 단원에서는 스레드와 관계 없이 독립적으로 생성되는 구성요소를 다룬다.

## 스레드

스레드는 프로그램의 실행 단위다. JVM은 애플리케이션이 여러 개의 스레드를 동시에 실행할 수 있도록 허용한다. HotSpot JVM에서는 자바 스레드와 네이티브 OS 스레드 사이에 직접적인 매핑이 존재한다. 자바 스레드의 상태를 구성하는 ThreadLocal 저장소, 버퍼 할당, 객체 동기화, 스택, 프로그램 카운터(PC)가 모두 준비된 후에 네이티브 스레드가 생성된다. 네이티브 스레드는 자바 스레드가 종료되면 회수된다. OS는 모든 스레드를 스케줄링하고 가용한 CPU로 스레드를 디스패치하는 일을 담당한다. 네이티브 스레드가 초기화되면 자바 스레드의 `run()` 메서드를 호출한다. `run()` 메서드가 반환되거나 잡히지 않은 예외가 처리되고 나면, 네이티브 스레드는 스레드의 종료와 함께 JVM도 종료되어야 하는지(즉, 스레드가 마지막 non-daemon 스레드인지) 확인한다. 스레드가 종료되면 네이티브 스레드와 자바 스레드에 사용된 모든 리소스는 반납된다.

### JVM 시스템 스레드

jconsole이나 어떤 디버거를 사용하면 백그라운드로 많은 스레드가 실행되는 것을 볼 수 있다. 이런 백그라운드 스레드는 `public static void main(String[])`를 호출하는 과정에서 생성되는 메인 스레드와 메인 스레드에 의해 생성되는 스레드에 덧붙여 함께 실행된다. HotSpot JVM에 있는 주요 백그라운드 시스템 스레드는 다음과 같다.

스레드 이름 | 설명
--- | ---
VM thread | JVM이 safe-point에 도달하게 만드는 연산을 기다리는 스레드. JVM이 safe-point에 도달하게 만드는 연산이 분리된 스레드에 발생해야 하는 이유는 그런 연산 모두가 JVM이 힙(heap)에 대한 수정이 발생할 수 없는 safe-point 상태가 되도록 This thread waits for operations to appear that require the JVM to reach a safe-point. The reason these operations have to happen on a separate thread is because they all require the JVM to be at a safe point where modifications to the heap can not occur. The type of operations performed by this thread are "stop-the-world" garbage collections, thread stack dumps, thread suspension and biased locking revocation.
Periodic task thread | 주기적으로 실행되는 연산을 스케줄링하는 데 사용되는 타이머 이벤트(즉, 인터럽트)를 담당하는 스레드.
GC threads | JVM에서 발생하는 유형별 가비지 컬렉션을 담당하는 스레드.
Compiler threads | 런타임에 바이트 코드를 네이티브 코드로 컴파일 하는 스레드.
Signal dispatcher thread | JVM 프로세스에 전달된 시그널을 받고 적절한 JVM 메서드를 호출해서 JVM 내에서 시그널을 처리하는 스레드.

### Per Thread

실행 스레드는 각각 다음의 구성요소를 가지고 있다.

#### 프로그램 카운터(Program Counter, PC)

현재 실행되는 네이티브가 아닌 instruction(또는 opcode)의 주소를 저장한다. 현재 메서드가 네이티브이면 PC는 정의되지 않는다. 모든 CPU는 PC를 가지고 있고, PC에 저장된 값은 일반적으로 각 instruction이 실행된 후에 증가하며, 다음에 실행될 instruction의 주소가 저장된다. JVM은 PC를 이용해서 instruction이 실행되는 위치를 추적한다. PC는 실제로 메서드 영역(Method Area)에 있는 메모리 주소를 가리킨다.

#### 스택(Stack)

각 스레드는 자기만의 스택을 가지며, 스택에는 해당 스레드에서 실행되는 메서드의 프레임(frame)이 저장된다. 프레임은 메서드가 새로 호출될 때마다 새로 생성되고, 스택은 후입선출(LIFO, Last In First Out)의 자료구조이므로 현재 실행되고 있는 메서드의 프레임이 스택의 맨 위에 위치한다. 프레임은 해당 메서드가 정상적으로 값을 반환하거나 메서드 실행 과정에서 잡히지 않은 예외가 던져지면 스택에서 꺼내어진다. 스택은 프레임 객체를 넣고 빼는 일을 제외하면 직접적으로 조작되지 않으며, 따라서 프레임 객체는 힙에 할당될 수도 있고 메모리는 연속적으로 인접해 있지 않아도 된다.

#### 네이티브 스택(Native Stack)

모든 JVM이 네이티브 메서드를 지원하는 것은 아니다. 하지만 네이티브 메서드를 지원하는 JVM은 스레드 별로 네이티브 메서드 스택을 생성한다. JVM이 JNI(Java Native Invocation)를 위해 C언어의 링크 모델을 사용해서 구현되었다면, 네이티브 스택은 C 스택이다. 이런 경우 인자와 반환값의 순서는 일반적인 C 프로그램의 네이티브 스택에서와 동일하다. 네이티브 메서드는 (JVM 구현에 따라 다를 수 있지만) 일반적으로 JVM 내부의 자바 메서드를 callback으로 호출한다. 이런 네이티브로부터 자바로 향하는 호출은 보통의 자바 스택 상에서 발생하며, 스레드는 네이티브 스택을 떠나서 보통의 자바 스택에 프레임을 생성해서 맨 위에 추가한다.

#### 스택 제약사항(Stack Restrictions)

새 프레임은 메서드 호출마다 새로 생성되어 스택의 맨 위에 추가되고, 메서드가 정상적으로 값을 반환하거나 메서드 실행 과정에서 잡히지 않은 예외가 던져지면 스택에서 꺼내어진다. 예외 처리는 아래의 [예외 테이블(Exception Tables)]()을 참고한다.

