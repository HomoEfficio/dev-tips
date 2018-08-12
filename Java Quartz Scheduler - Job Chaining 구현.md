# Java Quartz Scheduler - Job Chaining 구현

Java로 Job Scheduling을 쉽게(참 조심스러운 단어..) 할 수 있게 해주는 [쿼츠(Quartz) 스케줄러](http://www.quartz-scheduler.org/)가 있다.

이 사이트에 나와있는 문서들은 여태 본 기술 사이트 문서 중에 가장 맘에 드는 스타일로 구성되어 있다. 길지 않은 설명, 간략하면서도 필요한 정보를 모두 담고 있는 다양한 기본 예제와 실무형 Cookbook까지 정말 마음에 쏙 든다. 게다가 스프링부트 스타터로도 제공되므로 더욱 편리하게 프로젝트에서 사용할 수 있다.

그런데 옥의 티랄까.. 독립적인 Job은 훌륭한 문서와 쉬운 Fluent API 덕에 간단하게 구현할 수 있는데, 연속적인 Job 실행은 간단하게 구현할 수 있는 방법이 없는 것 같다. 그래서 검색을 해보니 결국에는 Job 실행에 사용되는 Context 객체 안에 다음에 실행할 Job을 넣어주고 스케줄링하는 방식으로 연속적인 Job 실행을 구현할 수 있다. 

그래서 간단하면서도 용도에 맞게 조금만 확장하면 아주 쓸만한 구현 예제를 만들어 봤다. 전체 코드는 https://github.com/HomoEfficio/quartz-scratchpad 에 있다.

## Quartz 기초 개념

쿼츠에 대한 감을 잡는 데는 단 한 줄이면 충분하다.

```java
scheduler.scheduleJob(jobDetail, trigger);
```

- `jobDetail`에는 Job의 실제 구현 내용과 Job 실행에 필요한 제반 상세 정보가 담겨 있다.
- `trigger`에는 Job을 언제, 어떤 주기로, 언제부터 언제까지 실행할지에 대한 정보가 담겨 있다.
- scheduler는 `jobDetail`과 `trigger`에 담긴 정보를 이용해서 실제 Job의 실행 스케줄링을 담당한다.

## Quartz 기초 예제

쿼츠 스케줄링을 통해 로그를 찍는 간단한 예제를 살펴보자.

### HelloJob

단순히 로그를 찍은 일을 하는 Job

```java
package io.homo.efficio.scratchpad.quartz;

import lombok.extern.slf4j.Slf4j;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-08-12
 */
@Slf4j
public class HelloJob implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("### Hello Job is being executed!");
    }
}
```

### QuartzTest

HelloJob을 스케줄링하고 실행하는 테스트. 물론 `public static void main()`으로 해도 무방하다.

```java
package io.homo.efficio.scratchpad.quartz;

import org.junit.Test;
import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.Trigger;
import org.quartz.impl.StdSchedulerFactory;

import static org.quartz.JobBuilder.newJob;
import static org.quartz.TriggerBuilder.newTrigger;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-08-12
 */
public class QuartzTest {

    @Test
    public void helloJob() throws SchedulerException, InterruptedException {

        // Job 구현 내용이 담긴 HelloJob으로 JobDetail 생성
        JobDetail jobDetail = newJob(HelloJob.class)
                .build();

        // 실행 시점을 결정하는 Trigger 생성
        Trigger trigger = newTrigger()
                .build();

        // 스케줄러 실행 및 JobDetail과 Trigger 정보로 스케줄링
        Scheduler defaultScheduler = StdSchedulerFactory.getDefaultScheduler();
        defaultScheduler.start();
        defaultScheduler.scheduleJob(jobDetail, trigger);
        Thread.sleep(3 * 1000);  // Job이 실행될 수 있는 시간 여유를 준다
        
        // 스케줄러 종료
        defaultScheduler.shutdown(true);
    }
}
```

### 테스트 결과

다음과 같이 HelloJob에 구현된 로그 출력이 성공적으로 수행된다.

```java
...
00:53:15.137 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.HelloJob - ### Hello Job is being executed!
...
```

## Job Chaining 기본 틀 구현

이제 위의 간단한 HelloJob을 넘어서 Job을 연속적으로 실행할 수 있는 Job Chaining을 구현해보자.

연속 실행 기능을 가질 추상 클래스인 `BaseJob`을 만들고, 실제 구현 내용을 담은 HelloJob은 `BaseJob`을 상속하게 만든다.

### BaseJob

```java
package io.homo.efficio.scratchpad.quartz;

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-08-12
 */
public abstract class BaseJob implements Job {

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        doExecute(context);
    }

    protected abstract void doExecute(JobExecutionContext context);
}
```

### HelloJob

```java
package io.homo.efficio.scratchpad.quartz;

import lombok.extern.slf4j.Slf4j;
import org.quartz.Job;
import org.quartz.JobExecutionContext;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-08-12
 */
@Slf4j
public class HelloJob extends BaseJob {

    @Override
    protected void doExecute(JobExecutionContext context) {
        log.info("### Hello Job is being executed!");
    }
}
```

### 테스트 재실행

테스트 코드는 바꿀 필요 없다. 실행해보면 전과 마찬가지로 로그가 성공적으로 출력된다.

```java
...
01:22:41.393 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.HelloJob - ### Hello Job is being executed!
...
```

### 템플릿 메서드 패턴 적용

`BaseJob`에 템플릿 메서드 패턴을 적용해서 Job 실행 `전처리`, `Job 실행`, `후처리`, `다음 Job Scheduling`이라는 파이프라인을 구성한다.

```java
package io.homo.efficio.scratchpad.quartz;

import lombok.extern.slf4j.Slf4j;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-08-12
 */
@Slf4j
public abstract class BaseJob implements Job {

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        beforeExecute(context);
        doExecute(context);
        afterExecute(context);
        scheduleNextJob(context);
    }

    private void beforeExecute(JobExecutionContext context) {
        log.info("%%% Before executing job");
    }

    protected abstract void doExecute(JobExecutionContext context);

    private void afterExecute(JobExecutionContext context) {
        log.info("%%% After executing job");
    }

    private void scheduleNextJob(JobExecutionContext context) {
        log.info("$$$ Schedule Next Job");
    }
}
```

### 테스트 재실행

테스트를 재실행해보면 다음과 같이 `전처리`, `Job 실행`, `후처리`, `다음 Job 스케줄링`이 실행됨을 알 수 있다.

```java
...
01:41:16.254 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% Before executing job
01:41:16.254 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.HelloJob - ### Hello Job is being executed!
01:41:16.254 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% After executing job
01:41:16.254 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - $$$ Schedule Next Job
...
```

## Job Chaining 실제 구현

여기에서는 쿼츠에 대한 부연 설명이 조금 필요하다.

`execute()` 메서드에 넘겨지는 `JobExecutionContext`에는 Job 실행에 필요한 다양한 정보를 담을 수 있다. 그 중에서도 `JobDataMap`을 이용하면 자유롭게 Key-Value 데이터를 담을 수 있다. 다음과 같이 테스트 코드를 바꿔서 정보를 담아보자.

```java
    @Test
    public void helloJob() throws SchedulerException, InterruptedException {

        // JobDataMap을 이용해서 원하는 정보 담기
        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put("JobName", "Job Chain 1");

        // Job 구현 내용이 담긴 HelloJob으로 JobDetail 생성
        JobDetail jobDetail = newJob(HelloJob.class)
                .usingJobData(jobDataMap)  // <- jobDataMap 주입
                .build();

        ... 이하 생략 ...
```

그리고 `HelloJob` 클래스도 `JobDataMap`에 담긴 정보를 사용하도록 바꿔보자.

```java
@Slf4j
public class HelloJob extends BaseJob {

    @Override
    protected void doExecute(JobExecutionContext context) {
        log.info("### {} is being executed!",
                context.getJobDetail().getJobDataMap().get("JobName").toString());
    }
}
```

테스트를 재실행하면 다음과 같이 `JobDataMap`에 담은 정보가 함께 출력된다.

```java
01:57:06.889 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% Before executing job
01:57:06.889 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.HelloJob - ### Job Chain 1 is being executed!
01:57:06.891 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% After executing job
01:57:06.891 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - $$$ Schedule Next Job
```

이제 `JobDataMap`에 다음 Job에 대한 정보를 담으면 Job Chaining을 할 수 있을 것 같다.

### Chaining 기본 아이디어

Job과 JobDataMap은 일대일 관계이므로, 

- Chaining 할 모든 Job 정보를 큐에 담고, 
- 그 큐를 처음 실행되는 Job의 `JobDataMap`에 담은 후에, 
- Job 실행이 완료되면 후처리 단계에서 실행이 완료된 Job을 큐에서 하나씩 빼주고,
- 다음 Job을 실행할 때 그 큐를 다음 Job의 `JobDataMap`에 넣어주고 스케줄링
- 큐가 비워지면 Chaining은 종료된다. 

### Chaining 할 여러 Job 생성

Job 3개를 Chaining해서 실행할 수 있도록 테스트 코드를 변경한다. 

예제에서는 편의상 3개의 Job에 모두 `HelloJob.class`만을 사용했지만, 실제로는 서로 다른 클래스를 사용해도 무방하다. 또한 [JobBuilder API](http://www.quartz-scheduler.org/api/2.2.1/org/quartz/JobBuilder.html)를 참고하면 Job마다 원하는 대로 식별자를 줄 수도 있고 오류 시 재실행 옵션 등 다양하게 설정할 수 있다. [TriggerBuilder API](http://www.quartz-scheduler.org/api/2.2.1/org/quartz/TriggerBuilder.html)를 참고하면 Trigger도 원하는 대로 더 다양하게 구성할 수 있다.

```java
public class QuartzTest {

    @Test
    public void helloJob() throws SchedulerException, InterruptedException {

        // Job 1 구성
        JobDataMap jobDataMap1 = new JobDataMap();
        jobDataMap1.put("JobName", "Job Chain 1");
        JobDetail jobDetail1 = newJob(HelloJob.class)
                .usingJobData(jobDataMap1)
                .build();

        // Job 2 구성
        JobDataMap jobDataMap2 = new JobDataMap();
        jobDataMap2.put("JobName", "Job Chain 2");
        JobDetail jobDetail2 = newJob(HelloJob.class)
                .usingJobData(jobDataMap2)
                .build();

        // Job 3 구성
        JobDataMap jobDataMap3 = new JobDataMap();
        jobDataMap3.put("JobName", "Job Chain 3");
        JobDetail jobDetail3 = newJob(HelloJob.class)
                .usingJobData(jobDataMap3)
                .build();
```

### Job 정보를 JobDataMap에 저장

실행할 모든 Job의 JobDetail를 첫 번째 JobDetail의 JobDataMap에 담는다.

```java
        // 실행할 모든 Job의 JobDetail를 jobDetail1의 JobDataMap에 담는다.
        List<JobDetail> jobDetailQueue = new LinkedList<>();
        jobDetailQueue.add(jobDetail1);
        jobDetailQueue.add(jobDetail2);
        jobDetailQueue.add(jobDetail3);
        // 주의사항: 아래와 같이 jopDataMap1에 저장하면 반영되지 않는다.
        // jobDataPam1.put("JobDetailQueue", jobDetailQueue);
        // 아래와 같이 jobDetail1에서 getJobDataMap()으로 새로 가져온 JobDataMap에 저장해야 한다.
        jobDetail1.getJobDataMap().put("JobDetailQueue", jobDetailQueue);
```

테스트 코드의 나머지 부분은 변경할 것이 없다.

나머지는 `BaseJob`에서 처리한다.

### 후처리 단계에서 완료된 Job을 큐에서 제거

`BaseJob`의 후처리 메서드인 `afterExecute()`를 다음과 같이 작성해서 큐에서 완료된 Job을 제거한다.

```java
    private void afterExecute(JobExecutionContext context) {
        log.info("%%% After executing job");
        Object object = context.getJobDetail().getJobDataMap().get("JobDetailQueue");
        List<JobDetail> jobDetailQueue = (List<JobDetail>) object;

        if (jobDetailQueue.size() > 0) {
            jobDetailQueue.remove(0);
        }
    }
```

### 다음 Job 스케줄링

`scheduleNextJob()` 메서드를 다음과 같이 변경해서, 완료된 Job이 제거된 큐를 `JobDataMap`에 담고 즉시 실행하는 Trigger를 만들어서 스케줄링 한다.

```java
    private void scheduleNextJob(JobExecutionContext context) {
        log.info("$$$ Schedule Next Job");
        Object object = context.getJobDetail().getJobDataMap().get("JobDetailQueue");
        List<JobDetail> jobDetailQueue = (List<JobDetail>) object;

        if (jobDetailQueue.size() > 0) {
            JobDetail nextJobDetail = jobDetailQueue.get(0);
            nextJobDetail.getJobDataMap().put("JobDetailQueue", jobDetailQueue);
            Trigger nowTrigger = newTrigger().startNow().build();

            try {
                // 아래의 팩토리 메서드는 이름이 같으면 여러번 호출해도 항상 동일한 스케줄러를 반환한다.
                Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
                scheduler.start();
                scheduler.scheduleJob(nextJobDetail, nowTrigger);
            } catch (SchedulerException e) {
                throw new RuntimeException(e);
            }
        }
    }
```

### 테스트 재실행

다음과 같이 Job 1, 2, 3이 모두 순차적으로 실행되는 것을 확인할 수 있다. 각 Job마다 서로 다른 워커 스레드에서 실행되는 것도 확인할 수 있다.

```java
...
02:33:47.650 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% Before executing job
02:33:47.650 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.HelloJob - ### Job Chain 1 is being executed!
02:33:47.651 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% After executing job
02:33:47.652 [DefaultQuartzScheduler_Worker-1] INFO io.homo.efficio.scratchpad.quartz.BaseJob - $$$ Schedule Next Job
...
02:33:47.655 [DefaultQuartzScheduler_Worker-2] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% Before executing job
02:33:47.656 [DefaultQuartzScheduler_Worker-2] INFO io.homo.efficio.scratchpad.quartz.HelloJob - ### Job Chain 2 is being executed!
02:33:47.656 [DefaultQuartzScheduler_Worker-2] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% After executing job
02:33:47.656 [DefaultQuartzScheduler_Worker-2] INFO io.homo.efficio.scratchpad.quartz.BaseJob - $$$ Schedule Next Job
...
02:33:47.658 [DefaultQuartzScheduler_Worker-3] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% Before executing job
02:33:47.658 [DefaultQuartzScheduler_Worker-3] INFO io.homo.efficio.scratchpad.quartz.HelloJob - ### Job Chain 3 is being executed!
02:33:47.658 [DefaultQuartzScheduler_Worker-3] INFO io.homo.efficio.scratchpad.quartz.BaseJob - %%% After executing job
02:33:47.659 [DefaultQuartzScheduler_Worker-3] INFO io.homo.efficio.scratchpad.quartz.BaseJob - $$$ Schedule Next Job
...
```

예제에서는 단순함을 위해 여러 연속적으로 실행될 Job을 관리하는 객체 없이 테스트 객체가 그 역할을 담당했지만, 실무에서는 예를 들면 `Batch` 같은 객체를 두고 그 안에 `List<Job>`을 둬서 책임 분리를 하는 것도 좋다.

## 정리

>쿼츠 스케줄러(Quartz Scheduler)는 문서화가 정말로 잘 되어 있고 API 설계도 잘 되어 있어서 정말 금방 익혀서 사용할 수 있다.
>
>다만, 연속적으로 Job을 실행할 수 있는 Job Chaining이 기본 사항으로 지원되지 않아 아쉽지만,
>
>템플릿 메서드 패턴을 적용하면 어렵지 않게 Job Chaining을 구현해서 적용할 수 있다.

