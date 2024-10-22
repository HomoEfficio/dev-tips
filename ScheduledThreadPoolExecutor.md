# ScheduledThreadPoolExecutor

- 일정 시간이 지난 후에 실행돼야 하는 작업이나
- 일정 주기로 반복 실행돼야 하는 작업 처리에 적합

## 특징

- 일반적인 ThreadPoolExecutor 는
  - corePoolSize 만큼 스레드를 채우고,
  - corePoolSize 를 넘어서는 작업은 Queue 에 채우고,
  - Queue 가 가득차면 maxPoolSize 만큼 스레드가 채워진다.
- 하지만 ScheduledThreadPoolExecutor 는
  >- **무제한 Queue 와 corePoolSize 만큼의 고정 크기를 가진 스레드풀처럼 동작한다.**
  - ![](https://i.imgur.com/lPMi7ja.png)
  - 일정 시간동안 실행되면 안 되고 무조건 대기를 해야하는 상황에 사용하므로 스레드 숫자를 늘리는 것보다는 큐에 계속 쌓는 것이 합당해 보인다.
  - 그런데 이럴거면 corePoolSize 와 maxPoolSize 를 같은 값으로 지정해서 생성하면 설명과도 딱 맞아떨어지므로 혼동이 없었을텐데,
  - 아쉽게도 다음과 같이 maxPoolSize 에 Integer.MAX_VALUE 를 지정해서 생성한다.
    ```java
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
    ```
  - 그래도 설명에 있는대로 무제한 큐에 작업이 쌓이고 스레드 갯수는 corePoolSize 이상 생성되지 않으므로,
  - 메모리 이슈가 있을지 검토해보고, 필요하다면 큐의 초기 사이즈인 16 대신에 더 큰 값을 지정하면 큐 사이즈 확장시 발생하는 복사에 의한 성능 손실을 예방할 수 있을 것 같은데,
  - ScheduledThreadPoolExecutor 와 그 안에 있는 내부 static 클래스인 DelayedWorkQueue 를 살펴보면 아쉽게도 초기 큐 사이즈를 지정할 수 있는 방법은 없어 보인다.

