# Java Native Memory Tracking

## DMA

자바에서도 `DirectBuffer`를 이용해서 JVM이 아닌 Native 메모리를 사용하고 DMA(Direct Memory Access)의 장점을 활용할 수 있다.

구체적인 사용법 등 자세한 내용은 [Java NIO Direct Buffer를 이용해서 대용량 파일 행 기준으로 쪼개기](https://homoefficio.github.io/2019/02/27/Java-NIO-Direct-Buffer를-이용해서-대용량-파일-행-기준으로-쪼개기/)를 참고하고 장단점만 요약하면 다음과 같다.

### 장점

- 디스크에 있는 파일을 운영체제 메모리로 읽어들일 때 CPU를 건드리지 않는다.
- 운영체제 메모리에 있는 파일 내용을 JVM 내 메모리로 다시 복사할 필요가 없다.
- JVM 내 힙 메모리를 쓰지 않으므로 GC를 유발하지 않는다.(물론 일정 크기를 가진 버퍼가 운영체제 메모리에 생성되는 것이고, 이 버퍼에 대한 참조 자체는 JVM 메모리 내에 생성된다)

### 단점

- DMA에 사용할 버퍼 생성 시 시간이 더 소요될 수 있다.
- 바이트 단위로 데이터를 취급하므로, 데이터를 행 단위로 취급하기 불편하다.
- 일반적인 Java 메모리 분석 방법으로는 추적할 수 없다.

요는 대용량 파일을 사용할 때 `DirectBuffer`를 사용하면 DMA의 장점을 누릴 수 있고 단점을 피할 수 있다.


## 메모리 사용 추적

그런데 JVM 메모리가 아닌 Native를 사용하므로 힙 덤프나 스레드 덤프 분석, `jstat` 등 일반적인 Java 메모리 분석 방법으로는 추적이 안 된다.

대용량 파일 사용 시 장점이 많다고 하니 아무래도 `DirectBuffer` 크기도 크게 잡을 수록 성능 상으로는 유리하겠지만, 그 큰 메모리가 어떻게 사용되고 회수되는지 확인이 안 된다면 곤란하다.

어쩌지?

뭘 어째.. 검색이지.. 검색해서 찾은 답은 `jcmd`다. 

간략하게 알아보자.


## Java 실행 옵션 추가

어떤 Java 애플리케이션의 Native 메모리 사용을 추적하려면 애플리케이션 실행 시 다음 옵션을 추가해줘야 한다.

>-XX:NativeMemoryTracking=summary


## 메모리 사용 현황 베이스라인 지정

이제부터 알아볼 메모리 사용 추적 방법은 **어떤 기준점 대비 메모리 사용량 증감(diff)을 기반으로 한다.** 따라서 먼저 비교 기준이 될 베이스라인(기준점)을 지정해준다.

>jcmd {PID} VM.native_memory baseline

위 명령을 실행하면 PID와 함께 `Baseline succeed`라는 짤막한 메시지만 출력된다. 비교 기준인 베이스라인이 지정됐다는 뜻이다.


## 메모리 사용 현황 출력 - 초기

이제 다음 명령을 실행하면 **메모리 사용 항목별로 베이스라인 대비 사용량 증감(diff)을 보여준다.**

>jcmd {PID} VM.native_memory summary.diff

지금까지 수행한 베이스라인 지정과 초기 현황 출력 결과는 다음과 같다.

![Imgur](https://i.imgur.com/SGbIKgm.png)

그리고 Native 메모리는 Internal 항목에 표시되며, 애플리케이션 실행 후 별다른 작업 없는 초기 상태에서 Native 메모리 사용량은 44KB 이다.

위 명령은 `jstat`처럼 주기적으로 실행하는 옵션은 없는 것 같고, 필요할 때마다 직접 실행하고 출력 내용을 확인하는 방식으로 진행한다.


## 메모리 사용 현황 출력 - DirectBuffer 사용 중

`DirectBuffer`를 사용하는 작업을 실행한 후에 다시 `jcmd {PID} VM.native_memory summary.diff`를 실행하면 다음과 같이 Internal 항목의 사용량이 기존 44KB에서 150MB로 대폭 늘어난 것을 확인할 수 있다.

![Imgur](https://i.imgur.com/t9gmDhx.png)

실제 코드에서도 다음과 같이 3개의 `DirectBuffer`를 각 50M 씩 할당했으므로 위 그림에서 출력된 내용과 잘 부합한다.

![Imgur](https://i.imgur.com/2AfnkJj.png)


## 메모리 사용 현황 출력 - DirectBuffer 사용 완료 후

`DirectBuffer`를 사용하는 작업 완료 후 별다른 조치 없이 다시 `jcmd {PID} VM.native_memory summary.diff`를 실행하면 다음과 같이 Internal 항목의 사용량이 150MB에서 328KB로 대폭 줄어든 것을 확인할 수 있다.

![Imgur](https://i.imgur.com/PRG5Nqh.png)

즉 `DirectBuffer` 사용이 끝난 후에 자동으로 Native 메모리가 정상적으로 회수됐음을 알 수 있다.


## 더 읽을 거리

- jcmd 명령 해설(공식 문서보다 훨씬 나음): https://www.javacodegeeks.com/2016/03/jcmd-one-jdk-command-line-tool-rule.html


## 정리

>- Java에서도 `DirectBuffer`를 이용해서 DMA를 활용할 수 있다.
>
>- DMA에 활용된 Native 메모리는 사용 완료 후 JVM GC와 무관하게(즉 다른 절차에 의해) 자동으로 반환된다.


