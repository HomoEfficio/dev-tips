# Blocking|Non-blocking vs Synchronous|Asynchronous

꽤 자주 접하는 용어다. 특히나 요즘들어 더 자주 접하게 되는데, 얼추 알고는 있고 알고 있는게 틀린 것도 아니지만, 막상 명확하게 구분해서 설명하라면 또 만만치가 않은..

그래서 찾아보면 또 대충 뭔소린지 알아들을 것 같다가도, 구분해서 설명하라면 머뭇거리게 되긴 마찬가지다.

자료마다 미세하나마 조금씩 차이가 있는 것들도 많아서, 정확하고 유일한 구분법은 사실 없는 것 같다. 그리고 이렇게라도 꼭 구분해야만 하는 것인가 하는 생각도 들지만, 그래도 '나의 언어로 구분하고 설명해보는 것'을 목표로 한 번 정리해보자.

# IBM developerWorks의 2:2 매트릭스

이 주제로 자료를 찾아본 사람들은 아마 아래의 그림을 한 번쯤은 봤을 것이다.

![IBM developerWorks의 단순화된 리눅스 I/O 모델 매트릭스](https://www.ibm.com/developerworks/library/l-async/figure1.gif)

위 그림만으로도 딱 와닿는다면 이미 Blocking-NonBlocking-Synchronous-Asynchronous의 구분에 대한 감이 있다고 볼 수 있으므로, 이 글은 더 이상 읽을 필요가 없다. ^^

와닿지 않는다면 조금 더 읽어보자. 하지만 앞으로 전개될 내용을 보면서, 이젠 너무 오래되어 버린 위의 IBM 그림(특히 I/O Multiplexing이 Blocking-Async로 구분되어 있는 점)에 구애 받을 필요는 없다. 

# 익숙한 것

일단 대충 알기로 Blocking과 Synchronous 둘이 비슷하고, Non-blocking과 Asynchronous 둘이 비슷하다. 그래서 대부분 아래 그림이 표시하는 개념을 알고 있다.

![Imgur](http://i.imgur.com/iSafBIF.png)

익숙해서 그런지 구체적인 예도 비교적 쉽게 떠올릴 수 있다.

![Imgur](http://i.imgur.com/06P0Q6m.png)

# 낯선 것

비슷해 보이는 것 둘씩 묶어보는 건 앞에서 살펴본 것처럼 꽤나 익숙하다. 문제는 비슷해 보이지 않는 걸 둘씩 묶어보는 건데, 이건 사례는 둘째치고 그림조차 잘 그려지지 않는다. 

먼저 Sync-NonBlocking을 생각해보자. 사실 Sync는 Blocking과 비슷한데, Blocking의 반대인 NonBlocking이랑 공존한다는 것 자체가 성립이 안되는 거 아닌가?

Async-Blocking도 마찬가지다. Async는 NonBlocking과 비슷한데, NonBlocking의 반대인 Blocking이랑 공존한다는 것 자체가 성립이 안되는 거 아닌가?

# 다른 관심사

Blocking-Sync가 비슷하고, NonBlocking-Async가 비슷하지만, Blocking/NonBlocking과 Sync/Async이 2:2 매트릭스 그림에서 각각 다른 축에 자리잡는 데는 이유가 있다. 두 그룹은 관심사가 다르다. 

## Blocking/NonBlocking 

>**Blocking/NonBlocking은 호출되는 함수가 바로 리턴하느냐 마느냐가 관심사**다. 

호출된 함수가 바로 리턴해서 호출한 함수에게 제어권을 넘겨주고, 호출한 함수가 다른 일을 할 수 있는 기회를 줄 수 있으면 NonBlocking이다.

그렇지 않고 호출된 함수가 자신의 작업을 모두 마칠 때까지 호출한 함수에게 제어권을 넘겨주지 않고 대기하게 만든다면 Blocking이다.

## Synchronous/Asynchronous

>**Synchronous/Asynchronous는 호출되는 함수의 작업 완료 여부를 누가 신경쓰냐가 관심사**다.

호출되는 함수에게 callback을 전달해서, 호출되는 함수의 작업이 완료되면 호출되는 함수가 전달받은 callback을 실행하고, 호출하는 함수는 작업 완료 여부를 신경쓰지 않으면 Asynchronous다.

호출하는 함수가 호출되는 함수의 작업 완료 후 리턴을 기다리거나, 또는 호출되는 함수로부터 바로 리턴 받더라도 작업 완료 여부를 호출하는 함수 스스로 계속 확인하며 신경쓰면 Synchronous다.

# 비슷한 동작, 다른 관심사

앞에서 막연하게 비슷하다고 했던 것은 조금 구체적으로 말하면 동작이 비슷한 것이었다. Blocking이나 Sync는 막거나 기다리거나 하는 등 둘 모두 뭔가 비효율적으로 동작하는 느낌인 반면에, NonBlocking이나 Async는 막지도 않고 완료되면 알아서 처리하는 등 둘 모두 뭔가 효율적으로 동작하는 느낌이다.

이제 동작은 비슷하지만 관심사가 다르다는 점을 염두에 두고 낯선 조합을 살펴보자.

## NonBlocking-Sync

앞에서 살펴본대로 조합해보면 NonBlocking-Sync는 호출되는 함수는 바로 리턴하고, 호출하는 함수는 작업 완료 여부를 신경쓰는 것이다. 신경쓰는 방법이 기다리거나 물어보거나 두 가지가 있었는데, NonBlocking 함수를 호출했다면 사실 기다릴 필요는 없고 물어보는 일이 남는다. 

즉, NonBlocking 메서드 호출 후 바로 반환 받아서 다른 작업을 할 수 있게 되지만, 메서드 호출에 의해 수행되는 작업이 완료된 것은 아니며, 호출하는 메서드가 호출되는 메서드 쪽에 작업 완료 여부를 계속 문의한다. 

그림을 그려보면 다음과 같다.

![Imgur](http://i.imgur.com/a8xZ9No.png)
 

이런 케이스가 뭐가 있을까 생각해보니 `future.isDone()`이 이것과 비슷한 것 같다.

```java
Future ft = asyncFileChannel.read(~~~);

while(!ft.isDone()) {
    // isDone()은 asyncChannle.read() 작업이 완료되지 않았다면 false를 바로 리턴해준다.
    // isDone()은 물어보면 대답을 해줄 뿐 작업 완료를 스스로 신경쓰지 않고,
    //     isDone()을 호출하는 쪽에서 계속 isDone()을 호출하면서 작업 완료를 신경쓴다.
    // asyncChannle.read()이 완료되지 않아도 여기에서 다른 작업 수행 가능 
}

// 작업이 완료되면 작업 결과에 따른 다른 작업 처리
```

참고로 위 코드는 NonBlocking-Sync라는 특성 이해에 집중할 수 있도록 간략화한 예제고, 실무적으로는 `Future`보다는 `CompletableFuture`를 쓰거나, `Future`를 쓴다면 위의 while 블록은 별도의 쓰레드로 빼서 실행하는 것이 좋다.

## Blocking-Async

이제 마지막 조각인 Blocking-Async다.

앞에서 살펴본대로 조합해보면 Blocking-Async는 호출되는 함수가 바로 리턴하지 않고, 호출하는 함수는 작업 완료 여부를 신경쓰지 않는 것이다.

그림을 그려보면 다음과 같다.

![Imgur](http://i.imgur.com/zKF0CgK.png)

이런 사례는 사실 생각해봐도 금방 떠오르는 게 없다. 어차피 Blocking되어 대기하는 것 외에는 다른 일도 못 하게된 마당에, 그냥 작업 끝날 때까지 기다렸다가 결과를 반환 받아서 처리하는 Blocking-Sync 방식과 성능적으로 거의 차이가 나지 않을 것 같은 방식이라서가 아닐까..

글을 올리고 보니 다른 분께서 좋은 의견을 주셨다. Blocking-Async는 별로 이점이 없어서 일부러 이 방식을 사용할 필요가 없기는 한데, 의도하지 않게 Blocking-Async로 동작하는 경우가 있다고 한다. 원래는 NonBlocking-Async를 추구하다가 의도와는 다르게 실제로는 Blocking-Async가 되어버리는 경우라고 하는데 그것은 바로..

**Blocking-Async의 대표적인 케이스가 Node.js와 MySQL의 조합**이라고 한다.

Node.js 쪽에서 callback 지옥을 헤치면서 Async로 전진해와도, 결국 DB 작업 호출 시에는 MySQL에서 제공하는 드라이버를 호출하게 되는데, 이 드라이버가 Blocking 방식이라고 한다. 

이건 사실 Node.js 뿐아니라 Java의 JDBC도 마찬가지다. 다만 Node.js가 싱글 쓰레드 루프 기반이라 멀티 쓰레드 기반인 Java의 Servlet 컨테이너보다 문제가 더 두드러져 보일 뿐, Blocking-Async라는 근본 원인은 같다.

그래서 Blocking-Async는 이렇게 정리해도 좋을 것 같다.

>Blocking-Async는 별다른 장점이 없어서 일부러 사용할 필요는 없지만, 
>
>**NonBlocking-Async 방식을 쓰는데 그 과정 중에 하나라도 Blocking으로 동작하는 놈이 포함되어 있다면 의도하지 않게 Blocking-Async로 동작**할 수 있다.


# 정리

>- **Blocking/NonBlocking은 호출되는 함수가 바로 리턴하느냐 마느냐가 관심사**
>    - 바로 리턴하지 않으면 Blocking
>    - 바로 리턴하면 NonBlocking
>    
>- **Synchronous/Asynchronous는 호출되는 함수의 작업 완료 여부를 누가 신경쓰냐가 관심사**
>    - 함수의 작업 완료를 호출한 함수가 신경쓰면 Synchronous
>    - 함수의 작업 완료를 호출된 함수가 신경쓰면 Asynchronous
>    
>- 성능 상으로 가장 유리한 모델은 Async-NonBlocking 모델이다.

![Imgur](http://i.imgur.com/gKDoKbs.png)

# 기타

공유하면 좋은 건 피드백을 받을 수 있다는 것이다.

Nonblocking과 Async를 관심사 관점이 아니라 다음과 같이 동작 관점에서도 구분할 수 있다는 좋은 의견도 있었다.

>- NonBlocking은 제어문 수준에서 지체없이 반환하는 것
>
>- Asynchronous는 별도의 쓰레드로 빼서 실행하고, 완료되면 호출하는 측에 알려주는 것

다음과 같이 입장(?)을 통해 구분한다는 의견도 좋아 보인다.

>- Blocking/NonBlocking은 호출한 입장에서의 특징
>- Sync/Async는 처리되는 방식의 특징

## I/O Multiplexing

아래 댓글에도 있지만, 페북 등 다른 채널을 통해서도 역시나 I/O Multiplexing에 대한 피드백이 많았다. 사실 어느 정도 예상을 하고 의도적으로 I/O Multiplexing에 대한 언급을 하지 않았는데, 그로 인해 오히려 의도하지 않은 부작용이 있는 것 같아서 지금이라도 언급을 하고 넘어가는 것이 좋을 것 같다.

결론부터 말하면, I/O Multiplexing이 IBM의 그림에서 Blocking-Async로 분류되어 있는 것에 동의하지 않는다.

일단 범위를 작게 해서 I/O Multiplexing 중에서 `select()` 호출 부분에만 국한해서 살펴보자.
[리눅스 man페이지의 select](http://man7.org/linux/man-pages/man2/select.2.html)의 description에는 아래와 같이 기술되어 있다.

>select() and pselect() allow a program to monitor multiple file
descriptors, **waiting** until one or more of the file descriptors become
"ready" for some class of I/O operation (e.g., input possible).  A
file descriptor is considered ready if it is possible to perform a
corresponding I/O operation (e.g., read(2) without blocking, or a
sufficiently small write(2))
>
>(대략) 하나 이상의 fd가 ready 상태가 될 때까지 wating, 즉, 기다린다. 

이렇게 보면 **OS 수준에서 `select()` 자체는 기다리는 Sync 방식**이라고 할 수 있다.

그럼 Blocking/NonBlocking은 어느쪽이냐 하는 문제가 남는데, 이는 구현에 따라 달라진다. Java를 예로 들면 API 문서에 [select()](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/Selector.html#select--)는 Blocking 방식이라고 기술되어 있으며, [selectNow()](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/Selector.html#selectNow--)는 NonBlocking 방식이라고 기술되어 있다.

따라서 **I/O Multiplexing 중에서 select의 호출만을 떼어서 말하면 일단 OS 수준에서 Sync 방식인 것은 자명하지만, Sync-Blocking인지 Sync-NonBlocking 인지는 구현 방식에 따라 달라지므로 어느 한 쪽이 맞다고 하기 어렵다**.

하지만, 보통 I/O Multiplexing이라고 하면 미시적으로 select의 호출 부분만을 말하는 것이 아니라, select의 대상이 되는 channel과의 연계까지도 포함하여 언급하려는 의도가 있다고 볼 수 있다. 그리고 이 과정에서의 channel은 NonBlocking 채널을 말한다. 

그 근거는 앞에서 인용한 리눅스 man 페이지의 내용 중에 '읽기에 대해 blocking이 없는 I/O 연산과, 충분히 작은 쓰기 I/O 연산을 수행할 수 있을 때, 해당 fd를 select 될 수 있는 ready 상태로 간주한다'는 내용이 있고, 실무적으로도 Java를 예로 들면 `channel.configureBlocking(false);`와 같이 채널을 NonBlocking으로 설정한다는 점이다.   

따라서 **일반적으로 I/O Multiplexing은 Sync방식의 select와 NonBlocking 방식의 channel이 연계되는 Sync-NonBlocking 방식이라고 해도 틀리지 않게 된다**.

# 더 읽을 거리

- https://www.lightbend.com/blog/7-ways-washing-dishes-and-message-driven-reactive-systems
- https://slipp.net/questions/367
- http://www.slideshare.net/unitimes/sync-asyncblockingnonblockingio
- http://djkeh.github.io/articles/Boost-application-performance-using-asynchronous-IO-kor/
- https://www.ibm.com/developerworks/library/l-async/


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
