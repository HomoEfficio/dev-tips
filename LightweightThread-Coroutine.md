# Lightweight Thread 와 Coroutine 관련 용어 교통 정리

#lightweight_thread, #coroutine, (Go)#goroutine, (JavaScript)#generator, (Java)#fiber, (Java)#virtual_thread, (C#)#async_await 는 #cooperative_multitasking(또는 #non_preemptive_multitasking)을 구현하는 방식이나 구현체를 지칭하는 용어들이라는 관점에서는 사실상 모두 같은 용어다. 맞나?


## Lightweight Thread

경량 스레드(Lightweight Thread)는 말 그대로 스레드인데 가벼운 스레드를 말한다. 스레드는 하나의 프로세스 안에서 자원을 할당 받아 작업을 수행하는 실행 흐름이다. 같은 프로세스에서 생성된 여러 스레드는 프로세스의 데이터 세그먼트, 코드 세그먼트, 파일을 공유하며, 실행 흐름 관리에 필요한 스택, 프로그램 카운터 등은 각 스레드 별로 따로 존재한다. 

![Imgur](https://i.imgur.com/kgBLalP.jpg)

그림 출처 - https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/4_Threads.html

경량 스레드도 스레드의 이런 특징을 그대로 가진다. 다만 가볍다는 점이 다르다.

뭐가 가볍다는 말일까? 크게 두 가지다. Context Switching 비용과 스레드 상태 유지를 위한 메모리 비용이 덜 든다는 얘기다. 컨텍스트 스위칭 비용은 결국 소요 시간이므로 이 비용이 낮으면 일정 시간 이내에 여러 스레드가 더 자주 자원을 할당 받아 더 많은 작업을 수행할 수 있다. 메모리 비용은 결국 메모리 점유량이므로 이 비용이 저렴하면 제한된 메모리 크기 안에서 더 많은 스레드가 생성될 수 있다.

그럼 왜 비용이 덜 드는 걸까? 경량 스레드는 커널 스레드가 아니기 때문이다. 커널 스레드를 사용하면 스레드 전환 시 실행 상태를 담고 있는 프로세서 레지스터의 보존과 교체라는 컨텍스트 스위칭이 발생하고 커널이 이 처리를 담당한다. 그런데 커널은 커널 스레드가 아닌 경량 스레드에 대해서는 전혀 알지 못한다. 그래서 하나의 프로세스 내의 하나의 커널 스레드에서 여러 개의 경량 스레드를 생성해서 사용하더라도 커널은 이 프로세스를 싱글 스레드 프로세스인 것으로 인식한다.

따라서 다수의 경량 스레드를 사용하더라도 커널 스레드 정보를 담고 있는 TCB(Thread Control Block)는 커널 스레드 갯수만큼만 필요하고, 경량 스레드 정보를 담고 있는 데이터 구조는 커널이 알아야 할 정보를 저장할 필요가 없으므로 TCB보다 메모리 사용량이 더 적다. 커널에 의한 컨텍스트 스위칭이 발생하지 않는 경량 스레드 사이에서의 전환은 결국 일반적인 함수 호출과 크게 다르지 않으므로 문맥 전환 비용도 훨씬 덜 든다.

경량 스레드는 커널 스레드가 아니고 커널에 의해 관리되지 않는다는 의미로 유저 레벨 스레드, 유저 랜드 스레드, 유저 모드 스레드 등으로 불리기도 한다. 

커널에 의해 관리되지 않는다는 것은 커널에 의해 강제로 선점되지 않는다는 것을 의미한다. 그래서 경량 스레드는 비선점형(Non-preemptive) 멀티태스킹 방식이라고 할 수 있다. 커널에 의해 선점되지 않으므로 점유한 자원을 애플리케이션 스스로 반환할 수 있어야 하며, 스스로 반환하는 것이 협력적이라는 의미에서 협럭적(Cooperative) 멀티태스킹 방식이라고 불리기도 한다. 


## Coroutine



## 

## 참고 자료

- https://wiki.c2.com/?VeryLightweightThreads
- http://www.cs.iit.edu/~cs561/cs450/ChilkuriDineshThreads/dinesh's%20files/User%20and%20Kernel%20Level%20Threads.html
- https://superuser.com/questions/669883/why-are-user-level-threads-faster-than-kernel-level-threads
- https://www.tutorialspoint.com/operating_system/os_multi_threading.htm
- https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/4_Threads.html
- https://www.cs.swarthmore.edu/~kwebb/cs45/s18/03-Process_Context_Switching_and_Scheduling.pdf
- https://nesoy.github.io/articles/2018-11/Context-Switching
- https://stackoverflow.com/a/2236484
- https://www.geeksforgeeks.org/thread-control-block-in-operating-system/
- https://slikts.github.io/concurrency-glossary/?id=system-threads-vs-user-level-threads-green-lightweight-threads
