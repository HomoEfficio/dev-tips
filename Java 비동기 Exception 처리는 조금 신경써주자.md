# Java 비동기 Exception 처리는 조금 신경써주자

별도의 스레드를 이용해서 비동기 방식으로 처리할 때, 동기 방식에서 하듯 무심코 예외 처리를 하면 예상치 않은 동작에 골탕을 먹을 수 있다.

먼저 익숙한 동기 방식에서의 예외 처리부터 살펴보자.

## Checked Exception 감싸 던지기

![Imgur](https://i.imgur.com/7dF9rvl.png)

`try-catch`로 예외를 잡은 후에 `throw new RuntimeException("메시지", e)`와 같이 `RuntimeException` 으로 감싸 던지면, 위와 같이 메시지와 StackTrace가 함께 출력 된다.

간단하면서도 원하는 바를 이룰 수 있으므로 자주 쓰이는 패턴이다.

맨 마지막에 exit code 도 255 라서 프로세스가 비정상 종료되었음을 알 수 있다.

## 다른 스레드 안에서 Checked Exception 감싸 던지기

![Imgur](https://i.imgur.com/znl4cHR.png)

비동기 방식에서는 슬슬 예상치 않은 동작이 나온다.

분명히 발생하는 예외인데 콘솔을 보면 아무 예외 메시지나 StackTrace도 없고 exit code 마저 0 이라서 정상 종료된 것처럼 보인다. 어떻게 된 걸까?

사실 이 케이스는 여러 번 실행을 하다 보면 아주 간간이 예외 메시지가 StackTrace가 보일 수도 있다.

프로그래밍에서는 되다 안 되다 하는 게 제일 골치 아프다.

아무런 오류 메시지도 없이 정상 종료된 것처럼 보이는 이유는 예외 메시지와 StackTrace가 출력되기도 전에 main 스레드가 종료되어 버리기 때문이다.

다음과 같이 main 스레드에서 충분한 시간 여유를 두면, 다른 스레드에서 출력한 예외 메시지와 StackTrace가 항상 표시된다.

![Imgur](https://i.imgur.com/1LXKdU0.png)

하지만 여전히 exit code는 0 이다. main 스레드가 정상 종료되면 다른 스레드의 정상 종료 여부와 관계 없이 항상 exit code는 0 으로 나오게 되어 있는 것 같다.


## 다른 스레드 안에서 발생한 Unchecked Exception 을 main 스레드에서 감싸 던지기

이번엔 Unchecked Exception 는 어떤지 알아보자. 다른 스레드에서 발생한 Unchecked Exception을 아무런 처리도 하지 않고 던져지게 두면 main 스레드에서 잡을 수 있을까?

![Imgur](https://i.imgur.com/Rx9Nf3F.png)

그냥 다른 스레드 내에서 예외 발생한 채로 상황이 종료되고 바깥의 main 스레드에는 던져지지 않는다.

`Thread.sleep()`으로 여유 시간을 줘도 마찬가지다. 

main 스레드에 던질 수 없다면, 다른 스레드 내에서 발생한 Unchecked Exception은 어떻게 처리해야 할까?


## 스레드 전용 예외 핸들러 적용

다음과 같이 `setUncaughtExceptionHandler()`를 이용해서 스레드 전용 예외 핸들러를 지정해 준다.

![Imgur](https://i.imgur.com/A7664PB.png)

스레드 전용 핸들러 안에 메시지 출력과 StackTrace 출력, 필요하다면 복구 로직을 구현하면 된다. 그러면 해당 스레드 안에서 예외 핸들러 로직을 실행한다.

위 캡쳐 코드에는 '잡아 던진다'는 표현을 썼지만 잘못된 표현이고, '발생한 예외를 처리한다'는 표현이 적절하다.

하지만 `setUncaughtExceptionHandler()`를 통해 지정된 예외 핸들러 안에서 발생한 예외는 아까와 마찬가지로 main 스레드에서는 잡히지 않는다. 그래서 메인 프로그램은 Exit code 0으로 종료된다.

![Imgur](https://i.imgur.com/MNt4MGK.png)

여기에서는 간단하게 스레드 전용 예외 핸들러만 알아봤지만, 시스템 내 모든 스레드에 적용할 수 있는 Default 예외 핸들러를 사용할 수도 있다. 물론 스레드 전용 예외 핸들러가 시스템 Default 예외 핸들러보다 우선 순위가 높다.

예외 핸들러 부분은 단순 스레드뿐 아니라 `Executors`에서도 같은 방식으로 처리할 수 있다.


## 정리

>다른 스레드 내에서 발생한 예외를 그 스레드 내에서 try-catch로 잡아 처리해도 콘솔에는 아무런 흔적도 안 남을 수 있다.
>
>다른 스레드 내에서 발생한 예외는 바깥 스레드로 던져지지 않는다.
>
>따라서 다른 스레드를 사용하는 비동기 방식에서는
>
>- 예외 메시지 출력과 StackTrace 출력을 명시적으로 처리해줘야 하고,
>
>- 스레드 전용 예외 핸들러를 사용하는 것이 좋다.
