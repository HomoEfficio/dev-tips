# Back to the Essence - Concurrency vs Parallelism

>동시성이 뭐냐?  
>복수의 태스크를 동시에 실행하는 거 아니냐?
>
>병렬성이 뭐냐?  
>복수의 태스크를 동시에 실행하는 거 아니냐?
>
>그럼 동시성과 병렬성이 뭐가 다른 거냐?  
>...

비슷하지만 다른 개념이라는 건 알겠는데, 설명하라면 또 명확하게 답하기가 쉽지 않다.

명확하게 답하기 쉽지 않은 이유는 몇 가지 관점에 따라 다르게 설명되어야 할 필요가 있는 것을 그냥 뭉뚱그려서 얘기해왔기 때문이다. 이제 관점에 따라 나눠서 살펴보자.


## 시간 관점

### Concurrency

일단 **동시성에서 말하는 동시는 물리적으로 완전히 동일한 한 시점만을 말하는 것이 아니라 사실 상 동시라고 간주할 수 있는 시점(virtually at the same time)도 포함**한다. 물리적으로 미세하게 다른 시점일지라도 애플리케이션 관점에서 동시라고 간주할 수 있는 시간 간격에서 복수의 태스크가 수행된다면 동시성이 있다.

그래서 **동시성은 CPU(또는 코어)가 1개인 상황에서도 가능**하다. 시분할 시스템을 통해 사실 상 동시라고 간주해도 무방한 시간에 여러 개의 태스크를 진행시키고 있다면 동시성이 있다.


### Parallelism

**병렬성에서 말하는 동시는 물리적으로 완전히 동일한 시점(physically and literally at the same time)** 이다.

그래서 **CPU(또는 코어)가 1개인 상황에서는 병렬성을 가질 수 없다.**


## 작업 독립성 관점

독립성은 관점에 따라 다르게 해석될 수 있지만, 그런 상대성을 용인하고 바라보면 다음과 같이 비교할 수도 있다.

### Concurrency

**동시성에서는 독립적인 복수 개의 태스크를 순서를 고려하지 않고 동시에 실행**한다.

1에서 100까지 더하는 태스크와 1에서 100까지 곱하는 태스크를 동시(완전한 동시가 아니라 동시성에서 말하는 동시)에 실행한다면 이 두 태스크는 독립적이며, 이런 태스크를 동시(완전한 동시가 아니라 동시성에서 말하는 동시)에 처리한다면 동시성이 있다.

순서를 고려하지 않는다는 것은 순서를 지키는 경우와 지키지 않는 경우 모두를 포괄한다.


### Parallelism

**병렬성에서는 하나의 태스크를 여러 부분으로 쪼개서 동시에 실행**한다.

1에서 100까지 더하는 하나의 태스크를 1-30, 31-50, 51-70, 71-100 이렇게 4개의 구간합으로 나누고, 이를 4개의 CPU(또는 코어)에서 각각 동시에 실행하면 병렬성이 있다.


## 동시성과 병렬성의 조합

### 동시성이 있으면서 병렬성은 없을 수 있다?

어떤 관점에서든 CPU(또는 코어)가 1개인 상황에서는 병렬성이 있을 수 없으므로, 이건 자명하다.


### 동시성은 없으면서 병렬성은 있을 수 있다?

위에서 다룬 더하기 구간합은 병렬성이 분명히 있지만, 동시성이 있다고 보는 것은 관점에 따라 다르다. 

구간합 자체를 별개의 태스크로 본다면 동시성이 있다고 볼 수 있고,
구간합을 별개의 태스크가 아니라 전체합이라는 하나의 태스크를 나눈 것으로만 본다면 동시성이 없다고 볼 수 있다.


### 동시성과 병렬성 모두 있을 수 있다?

1에서 100까지 더하는 하나의 태스크를 1-50, 51-100 이렇게 2개의 구간합으로 나누고, 이를 1, 2번 CPU(또는 코어)에서 각각 동시에 실행하고, 동시에 1에서 100까지 곱하는 하나의 태스크를 1-50, 51-100 이렇게 2개의 구간곱으로 나누고, 이를 3, 4번 CPU(또는 코어)에서 각각 동시에 실행한다면 어떨까?

이럴 때는 더하기와 곱하기라는 독립적인 복수의 태스크를 동시에 실행하므로 동시성이 있고,  
더하기라는 하나의 태스크를 구간합으로 나눠서 동시에 실행하고, 곱하기라는 하나의 태스크를 구간곱으로 나눠서 동시에 실행하므로 병렬성도 있다.


## concurrent vs simultaneous

참고로 전산 용어를 잠시 떠나서 concurrent와 simultaneous를 비교해보는 것도 concurrent의 의미를 이해하는 데 도움이 될 것 같다.

둘의 차이를 아래와 같이 해석하는 게 100% 맞는지는 원어민이 아닌 나는 알 수 없지만, 최소한 concurrency vs parallelism을 이해하는 데는 확실히 도움이 된다. 어설픈 번역 말고 그냥 원어 그대로 가져와본다. 출처는 [Quora](https://www.quora.com/Is-there-any-major-difference-between-simultaneous-and-concurrent)다.

![Imgur](https://i.imgur.com/2KAJ16s.png)


## Go 언어를 만든 Rob Pike의 강연

Rob Pike는 [강연](https://www.youtube.com/watch?v=cN_DpYBzKso)에서 Concurrency와 Parallelism는 다르다며 여러 주옥 같은 표현을 남겼다.

>동시성은 독립적인 실행 프로세스의 조합을 의미하고,
>병렬성은 여러 가지 일을 동시에 실행하는 것을 의미한다.
>
>`Concurrency is the composition of independently executing processes.`
>`Parallelism is the simultaneous execution of multiple things.`

참고로 이 말에 나온 프로세스는 리눅스 프로세스가 아닌 스레드, 코루틴을 포괄하는 일반적인 의미의 프로세스를 의미한다.

>동시성은 여러 가지 일을 한 번에 **처리**하는 것을 말하고,  
>병렬성은 여러 가지 일을 한 번에 **수행**하는 것을 말한다.
>
>`Concurrency is about **dealing with** lots of things at once.`  
>`Parallelism is about **doing** lots of things at once.`

`dealing with`와 `doing`이 영어로는 확연하게 대조되지만, 간결함과 글자수 맞춤을 위해 `처리`와 `수행`으로 옮겨보면 그 차이가 좀 희석되는 것 같다.  
`dealing with`에는 설계가 포함되고, `doing`은 설계된 대로 수행하는 것을 의미한다고 이해하면 크게 틀리지 않을 것 같다.

>동시성은 **구조**에 관한 것이고,  
>병렬성은 **실행**에 관한 것이다.
>
>`Concurrency is about the **structure**.`  
>`Parallelism is about the **execution**.`

오호 이게 좋다. 영어든 국어든 글자수까지 똑같아서 아름답기까지 하다..  
이렇게 보면 Parallelism을 병렬성보다는 병행성으로 옮기는 게 더 나은 것 같다.  

정리하면 Rob Pike는 Concurrency는 설계 쪽에 무게를 두고, Parallelism은 실행 쪽에 무게를 두는 것 같다.

## 그림 비교

간단하게는 썸네일로 사용한 이 그림도 괜찮고,

![Imgur](https://i.imgur.com/cDdWLKL.jpg)

(출처: https://www.codeproject.com/Articles/1267757/Concurrency-vs-Parallelism)

다음 그림도 괜찮은 것 같다.

![Imgur](https://i.imgur.com/uIMnkj1.jpg)

(출처: https://twitter.com/ohidxy/status/946110898539659264)

## 마무리

Concurrency와 Parallelism을 구별할 때 다음과 같은 해석이 도움이 된다.

- '동시'의 차이
  - Concurrency에서 말하는 동시성은 사실 상 동시라고 간주해도 되는 시간 간격을 의미
  - Parallelism에서 말하는 동시성은 완전히 동일한 시점을 의미

- 작업 독립성의 차이
  - Concurrent하게 실행되는 작업은 일반적으로 서로 독립적
  - Parallel하게 실행되는 작업은 일반적으로 원래 하나인 작업을 동시에 실행할 수 있도록 분할한 작업을 의미

Go 언어를 만든 Rob Pike에 의하면,

- Concurrency는 독립적인 여러 작업을 한 번에 수행하도록 설계하는 쪽에 무게를 둔 개념
- Parallelism은 하나의 작업을 설계된 대로 분할해서 동시에 수행하는 쪽에 무게를 둔 개념


### 참고 자료

- https://howtodoinjava.com/java/multi-threading/concurrency-vs-parallelism/
- https://www.slideshare.net/PramestiHattaK/golang-101-concurrency-vs-parallelism
- http://tutorials.jenkov.com/java-concurrency/concurrency-vs-parallelism.html
- https://www.amazon.com/Reactive-Programming-RxJava-Asynchronous-Applications/dp/1491931655/


## 추가

페북에 공유하고 보니 이규원 님으로부터 다음과 같은 아주 쌈박한 의견을 얻을 수 있었다.

>Concurrency는 다수의 문제가 동시에 일어난 상황에 대한 것이고,  
>Parallelism은 다수의 문제를 동시에 해결하는 방법에 대한 것입니다.

개인적으로는

>Concurrency는 해결해야 할 문제고,
>Parallelisma은 해결하는 방법이다.

라고 생각해왔는데, 몇 군데 조사해보니 이런 식으로 서술된 게 없어서 아닌가보다.. 하고 Rob Pike의 설명대로 구조 vs 실행에 힘을 실어주고 끝맺었는데, 우군을 얻었으니 Rob Pike와는 조금 다를지라도 다음과 같이 결론낸다.

>**Concurrency는 동시에 발생한 다수의 일을 처리해야하는 상황을 의미**하고,  
>**Parallelism은 다수의 일을 동시에 실행하는 방식을 의미**한다.
