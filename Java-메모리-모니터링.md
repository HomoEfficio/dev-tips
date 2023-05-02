# Java 메모리 모니터링

# Heap Dump

>힙에 있는 모든 객체 Dump  
>jmap -dump:format=b,file=HEAP_DUMP_OUTPUT_FILE_NAME.hprof PID
>
>힙에 있는 Live 객체만 Dump  
>jmap -dump:live,format=b,file=HEAP_DUMP_OUTPUT_FILE_NAME.hprof PID

# 힙 분석 도구

jmap 으로 생성한 Heap Dump 파일을 열어서 분석할 수 있는 도구

- Eclipse MAT: https://www.eclipse.org/mat/
  - 메모리 누수 의심 내역을 보고서 형태로 보여줘서 좋음
  - ![Imgur](https://i.imgur.com/zeiXYxF.png)
  - ![Imgur](https://i.imgur.com/yEa9Xpk.png)
  - ![Imgur](https://i.imgur.com/xiTL9Lh.png)
  - Shallow Heap vs Retained Heap: https://dzone.com/articles/eclipse-mat-shallow-heap-retained-heap
    - shallow heap은 한 객체만이 점유한 힙의 크기
    - retained heap은 한 객체가 제거될 때 함께 제거될 수 있는 객체들이 점유하고 있는 힙의 크기
- VisuamVM: https://visualvm.github.io/

# 메모리 사용 및 GC 모니터링

>jstat -gc -h10 -t PID 10000  
>10줄마다 헤더 출력, 타임스탬프 출력, 10000밀리초마다 출력

![Imgur](https://i.imgur.com/d0TMbpm.png)

# Thread Dump

>jstack PID > THREAD_DUMP_OUTPUT_FILENAME

# Thread Dump 분석 사이트

- https://fastthread.io

# 자바 메모리 분석 관련 좋은 자료

- GC 관련: 
  - https://d2.naver.com/helloworld/6043
- OOM 및 Heap Shrinkage 관련: 
  - https://www.samsungsds.com/global/ko/support/insights/1209174_2284.html
- 메모리 누수 관련: 
  - https://d2.naver.com/helloworld/1326256
  - https://woowabros.github.io/tools/2019/05/24/jvm_memory_leak.html
- Thread Dump 관련:
  - https://brunch.co.kr/@springboot/126
  - https://d2.naver.com/helloworld/10963

# 짤막 지식

## Heap Shrinkage

- **메모리 사용량을 증가 시키는 작업이 끝난다고 해서 메모리 사용량이 바로 줄어들지는 않음**
    - **어떤 작업 완료 후 top로 확인한 메모리 사용이 줄지 않는다고 해서 무조건 메모리 누수라고 판단하면 안됨**

### 사례

- 작업 시작 전
    - ![Imgur](https://i.imgur.com/oZV4Eat.png)
- 작업 중 - 메모리 사용량 증가
    - ![Imgur](https://i.imgur.com/t8hwQHz.png)
- 작업 완료 후 - 메모리 사용량 감소하지 않음
    - ![Imgur](https://i.imgur.com/sGBCLbK.png)
- 메모리 회수 - jmap 자체가 메모리 회수 명령은 아니지만, 부수적으로 메모리 사용량이 감소함
    - ![Imgur](https://i.imgur.com/gQze3QN.png)
- 따라서 jmap 을 힙 덤프 뿐아니라 메모리 누수 확인 간편법으로 사용할 수도 있음
    - 반드시 `-dump:live` 옵션을 줘야 과도한 크기의 힙 덤프 파일이 생성되지 않음
    - jmap 을 해도 메모리 사용량이 많이 줄지 않는다면 메모리 누수가 있는 걸로 추정 가능

## 경험상 중첩된 Collection은 메모리 이슈 발생할 낌새가 좀 있더라..

- List의 원소 안에 또 List 가 있고 그 안에 또 List 가 있고..
- Map 의 value 중에 또 Map 이 있고 그 안에 또 Map 이 있고..
- 대량의 데이터가 위와 같은 형태로 처리되면 메모리 회수 타이밍이 좀 지연되는 것 같고, 결국 메모리 사용량이 과도하게 높아져서 OOM 이 발생하기도 하더라..
- 이럴 때는 GC에 맡기지 말고 중간중간 Collection 안에 중첩된 Collection 들을 직접 `clear()` 해주는 게 도움이 될 수도..
- 이 경우 JPA 라면 LazyLoading을 통해 OOM 을 방지할 수도 있다

