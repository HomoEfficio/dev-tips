# Quartz 스케줄러 적용 아키텍처 개선 - 1

Quartz는 스프링에서도 지원하고 있어서 스프링 기반 프로젝트에도 쉽게 통합해서 사용할 수 있으므로 널리 사용되고 있다. 게다가 properties 파일 설정만으로 간단하게 클러스터를 구성해서 부하 분산 및 fail-over가 가능한 것도 장점이다.

Quartz에 나오는 주요 등장 인물에 대한 설명이나 기본 사용 방법은 검색해보면 많이 나오므로 여기에서는 생략하고 실제 적용할 때 필요한 아키텍처 개선에만 초점을 맞춘다. 관련 코드 역시 주요 부분만 표시하면서 진행한다.

OpenJDK 9, 그레이들 5.6.2, 스프링부트 2.1.8, 스프링 5.1.9, Quartz 2.3.1 기준이다.

# Quartz 실행 흐름

Quartz 실행 흐름은 다음 그림에 잘 정리돼 있다. 간단하게 요약하면 `Scheduler`가 `Job`의 실행 정보를 통해 정해진 시간이나 정해진 주기로 `Job`을 실행한다.

![Imgur](https://i.imgur.com/JB5c5mF.jpg)

그림 출처: https://examples.javacodegeeks.com/enterprise-java/quartz/java-quartz-architecture-example/

# 문제

그림에서 보듯 실행 흐름 관점에서 보면 `Scheduler`가 `Job`을 사용하며 `Job`에 의존하고 있다. 하지만 `Scheduler`는 사실 상 Quartz 프레임워크 그 자체로서 안정적인 모듈이고, `Job`은 `Scheduler`에 의해 실행될 실제 작업에 대한 로직이 포함돼 있어 변경 빈도가 더 높고, `Job` 자체가 추가/삭제되는 일도 빈번하므로 불안정하다. 안정적인 모듈이 불안정한 모듈에 의존하는 것은 좋지 않다. Quartz 개발자들이 이런 사실을 모를 리 없다.

그래서 `Job`은 사실 인터페이스다. 그리고 `Scheduler`에 의해 실행되는 실제 작업들은 `Job` 인터페이스의 구현체 들이다. 즉 Quartz 자체는 `Scheduler`와 `Job` 구현체를 분리할 수 있게 잘 설계돼 있다.

그런데 이렇게 **분리할 수 있음에도 불구하고 `Scheduler`와 `Job` 인터페이스의 구현체를 같은 서버 안에 두면서 문제가 시작된다.** 작업을 추가하려면 스케줄러까지 재배포를 해야 되며, 스케줄러 재배포는 스케줄러의 중단을 의미하므로 실행되는 작업이 없을 때만 가능하며, 많은 작업이 스케줄링 돼 있다면 재배포 타이밍을 잡기가 어려워진다. 따라서 스케줄러 본체인 `Scheduler`와 작업 클래스인 `Job` 구현체를 분리해서 별도로 배포할 수 있으면 이 문제를 해결할 수 있다.

하지만 인터넷에서 찾을 수 있는 대부분의 Quartz 사용법이나 예제는 `Scheduler`와 `Job` 구현체를 동일한 서버 인스턴스에 일체형으로 묶어둔 아키텍처를 기준으로 설명하고 있다. 아마도 Quartz의 사용법 자체에 중점을 두기 때문이겠지만, 이유야 어찌됐든 자료가 그러하므로 결국 실제 프로젝트에 적용할 때도 일체형으로 구성하는 곳이 많은 것 같다.

이제 스케줄러와 작업 클래스를 분리해서 별도로 배포할 수 있게 만들고 클린 아키텍처에 한 걸음 다가가보자. 3가지 고개를 넘어야 한다.

# 클래스로딩

스케줄러가 작업을 스케줄하려면 작업 클래스를 로딩하고 실행 주기를 지정해야 한다. 대략 다음과 같다.

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class InitRunner implements CommandLineRunner {

    private final Scheduler scheduler;


    @Override
    public void run(String... args) throws Exception {
        log.info("Init Runner executed.");
        JobKey jobKey = JobKey.jobKey("jobkey1", "jobgroup1");
        JobDetail jobDetail = buildJobDetail(jobKey);
        Trigger trigger = buildJobTrigger(jobKey);
        scheduler.scheduleJob(jobDetail, trigger);  // (1)
    }

    private JobDetail buildJobDetail(JobKey jobKey) throws ClassNotFoundException {
        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put("key1", "value1");
        jobDataMap.put("key2", 2);
        
        return JobBuilder.newJob(SimpleJob.class)  // (2)
                .withIdentity(jobKey)
                .withDescription("Simple Quartz Job Detail")
                .usingJobData(jobDataMap)
                .build();
    }

    private Trigger buildJobTrigger(JobKey jobKey) {
        return TriggerBuilder.newTrigger()
                .forJob(jobKey)
                .withDescription("Simple Quartz Job Trigger")
                .startNow()  // 스케줄링 되면 바로 실행되는 방식
                .build();
    }
}


@Slf4j
public class SimpleJob implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("OOO JOB [{}] executed.", this.getClass().getSimpleName());
        JobDataMap mergedJobDataMap = context.getMergedJobDataMap();
        mergedJobDataMap.forEach((k, v) -> log.info("OOOOO {}: {}", k, v));
    }
}
```

스케줄러에 의해 실행되는 `SimpleJob`은 단순하게 `JobExecutionContext`에 담겨 있는 키-밸류를 출력한다.

(1)과 같이 `JobDetail`과 `Trigger`를 `Scheduler`에 전달해주면 스케줄링 된다. 실행될 작업의 클래스는 (2)와 같이 클래스 리터럴 형태로 `JobDetail`에 지정된다. `Trigger`는 편의상 (3)과 같이 바로 실행되는 방식으로 지정했지만 Cron Expression 등 다양한 방식으로 지정할 수 있다.

실행해보면 다음과 같이 `SimpleJob`이 실행된다.

![Imgur](https://i.imgur.com/XDItlAj.png)

## 커스텀 클래스로더

아키텍처 관점에서 중요한 지점은 (2)다. 일체형일 때는 위와 같이 직접 `SimpleJob.class`로 지정해주면 되지만, 분리돼 있다면 해당 클래스를 외부에서 읽어와야 한다. 따라서 다음과 같이 `URLClassLoader`를 활용해서 `JOB_REPO`로 지정된 jar 파일에 있는 작업 클래스를 로딩할 수 있는 커스텀 클래스로더가 필요하다.

```java
@Component
@Slf4j
public class RemoteJobClassLoader {

    private static final String JOB_REPO = "/tmp/homo-efficio/quartz/remote-job-repo/quartz-job.jar";

    @SuppressWarnings("unchecked")
    public <T> Class<? extends T> loadClass(String name, Class<T> clazz) throws ClassNotFoundException {
        return (Class<? extends T>) getClassLoader().loadClass(name);
    }

    private ClassLoader getClassLoader() {
        try {
            return new URLClassLoader(
                    new URL[] {
                            new File(JOB_REPO).toURI().toURL()
                    },
                    // URLClassLoader 설정 시 parent를 webAppClassLoader로 지정해줘야
                    // org.quartz.Job 등 내부 의존 클래스 로딩 가능
                    this.getClass().getClassLoader()
            );
        } catch (MalformedURLException e) {
            throw new RuntimeException(e);
        }
    }
}

```

자바 클래스로딩은 https://homoefficio.github.io/2018/10/13/Java-클래스로더-훑어보기/ 와 https://homoefficio.github.io/2018/10/14/Java-URLClassLoader로-알아보는-클래스로딩/ 를 보면 도움이 될 것이다.

## 작업 클래스 외부화

이제 작업 클래스를 외부로 분리해보자. 지금까지 만든 파일은 모두 `quartz-scheduler`에 있고, 새로 `quartz-job` 모듈을 만들어서 작업 클래스는 모두 이 모듈에 둔다.

![Imgur](https://i.imgur.com/4BFS4lx.png)

`quartz-job` 모듈의 `build.gradle` 파일은 다음과 같이 작성한다. jar 파일을 `RemoteJobClassLoader`가 읽을 수 있는 위치에 배포하는 `deployJar` 태스크를 추가했다.

```
plugins {
    id 'java'
}

version 'unspecified'

sourceCompatibility = '9'

repositories {
    mavenCentral()
}

dependencies {
    compileOnly 'org.projectlombok:lombok:1.18.8'
    annotationProcessor 'org.projectlombok:lombok:1.18.8'
    compile group: 'org.quartz-scheduler', name: 'quartz', version: '2.3.1'
    compile group: 'org.springframework', name: 'spring-context-support', version: '5.1.9.RELEASE'

    testCompile group: 'junit', name: 'junit', version: '4.12'
}

task deployJar(type: Copy) {
    dependsOn('jar')
    from "$buildDir/libs/quartz-job.jar"
    into "/tmp/homo-efficio/quartz/remote-job-repo/"
}
```

아까 나왔던 `SimpleJob`과 거의 동일한 내용의 `RemoteSimpleJob` 클래스를 `quartz-job` 아래에 만든다.

```java
@Slf4j
public class RemoteSimpleJob implements Job {

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("OOO REMOTE JOB [{}] executed.", this.getClass().getSimpleName());
        JobDataMap mergedJobDataMap = context.getMergedJobDataMap();
        mergedJobDataMap.forEach((k, v) -> log.info("OOOOO {}: {}", k, v));
    }
}
```

작업 스케줄링 하는 부분을 다음과 같이 `RemoteJobClassLoader`를 사용해서 작업 클래스를 로딩하도록 바꾼다.

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class InitRunner implements CommandLineRunner {

    private final Scheduler scheduler;
    private final RemoteJobClassLoader remoteJobClassLoader;  // 여기!!

    @Override
    public void run(String... args) throws Exception {
        log.info("Init Runner executed.");
        JobKey jobKey = JobKey.jobKey("jobkey1", "jobgroup1");
        JobDetail jobDetail = buildJobDetail(jobKey);
        Trigger trigger = buildJobTrigger(jobKey);
        scheduler.scheduleJob(jobDetail, trigger);
    }

    private JobDetail buildJobDetail(JobKey jobKey) throws ClassNotFoundException {
        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put("key1", "value1");
        jobDataMap.put("key2", 2);
        
        // 여기!!
//        return JobBuilder.newJob(SimpleJob.class)
        return JobBuilder.newJob(remoteJobClassLoader.loadClass("io.homo_efficio.quartz.job.RemoteSimpleJob", Job.class))
                .withIdentity(jobKey)
                .withDescription("Simple Quartz Job Detail")
                .usingJobData(jobDataMap)
                .build();
    }

    private Trigger buildJobTrigger(JobKey jobKey) {
        return TriggerBuilder.newTrigger()
                .forJob(jobKey)
                .withDescription("Simple Quartz Job Trigger")
                .startNow()
                .build();
    }
}
```

이제 다시 애플리케이션을 실행해보면 다음과 같이 분리된 별도의 jar 파일에서 작업 클래스를 로딩해서 실행하는 것을 확인할 수 있다.

![Imgur](https://i.imgur.com/gF10uft.png)

스케줄러쪽에서 `io.homo_efficio.quartz.job.RemoteSimpleJob`와 같이 외부 jar에 있는 작업 클래스 위치를 문자열로 직접 참조하고 있어서 마치 스케줄러 모듈(quartz-scheduler)이 작업 모듈(quartz-job)에 의존하는 것처럼 보이지만, 실무에서는 작업 클래스 위치나 실행 주기 정보를 DB에서 읽어오므로 실제 환경에서는 스케줄러 모듈은 작업 클래스 모듈을 모른다. 따라서 이제부터는 `RemoteJob2`, `RemoteJob3` 등을 추가하더라도 `quartz-job.jar`만 빌드/배포하면 되며, 스케줄러는 재배포할 필요가 없는 구조가 만들어졌다.

이렇게 해서 스케줄러와 작업 클래스를 분리하는데 성공했다. 어렵지 않다. 

그런데 실무에서 사용하는 작업 클래스들이 `RemoteSimpleJob`처럼 단순할리는 없다. DB 작업도 있을 것이고 하둡 인프라와 관련한 작업도 있을 것이다. 이런 것들이 가능하려면 스케줄러 쪽에 있는 컴포넌트를 `@Autowire`로 주입 받아야 하는데, 지금처럼 런타임에 로딩되는 방식에서도 가능할까?

첫번째 고개만으로도 양이 제법되니 일단 여기서 1탄을 마무리하고 `@Autowire`는 [2탄](https://homoefficio.github.io/2019/09/29/Quartz-스케줄러-적용-아키텍처-개선-2/)에서 다룬다.

# 정리

>- 스케줄링을 담당하는 스케줄러와 실행되는 작업은 변경 주기가 다르다. 그런데 이를 한 곳에 모아 일체형으로 구성하면 운영이 매우 불편해진다.
>
>- Quartz는 스케줄러와 작업을 분리할 수 있도록 설계되어 있다.
>
>- 작업 클래스를 별도의 jar로 묶고, 스케줄러 쪽에서 `URLClassLoader`를 사용해서 작업 클래스를 로딩하도록 개선하면 스케줄러와 작업 클래스의 배포를 분리할 수 있어 운영 부담을 대폭 줄일 수 있다.
