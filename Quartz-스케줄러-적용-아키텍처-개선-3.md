# Quartz 스케줄러 적용 아키텍처 개선 - 3

[1편](https://homoefficio.github.io/2019/09/28/Quartz-스케줄러-적용-아키텍처-개선-1/)에서는 Quartz 스케줄러 적용 시 변경 주기가 다른 스케줄러 모듈과 작업 클래스 모듈을 분리해서 클린 아키텍처에 다가가는 방법을 알아봤다.

[2편](https://homoefficio.github.io/2019/09/29/Quartz-스케줄러-적용-아키텍처-개선-2/)에서는 Quartz 작업 클래스 모듈에 의존 관계를 주입하는 방법을 알아봤다.

이렇게 두 개의 고개를 성공적으로 넘었고 마지막으로 `@Tranactional` 고개가 남았다.

먼저 일반적인 상황, 즉 스프링부트 애플리케이션인 스케줄러 모듈에서 `@Transactional`을 사용하는 간단한 코드를 추가해서 `@Transactional`의 동작을 확인하고, 작업 클래스 모듈에서 `@Tranactional`을 사용해보자.


# 스케줄러 모듈에서의 `@Transactional` 동작 확인

`@Transactional` 동작을 확인하기 위해 스케줄러 모듈에 `Member` 엔티티와 JPA 리포지토리를 추가한다.

```java
@Entity
@Getter
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String email;

    protected Member() {
    }

    public Member(String name, String email) {
        this.name = name;
        this.email = email;
    }
}
```

```java
@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {

}
```

`HelloService`에 `@Transactional`을 사용하는 메서드를 추가한다. 저장 후 일부러 예외를 발생시켜서 롤백되게 한다.

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class HelloService {

    private final MemberRepository memberRepository;

    public void sayHello() {
        log.info("OOO {}.sayHello() executed", this.getClass().getSimpleName());
    }

    @Transactional
    public Member saveMember(Member member) {
        Member dbMember = memberRepository.save(member);
        if (1==1) {
            throw new RuntimeException("테스트를 위해 강제로 발생시킨 예외");
        }
        return dbMember;
    }
}
```

`InitRunner`에 다음과 같이 `HelloService.saveMember()`를 호출하고 예외를 잡아 처리하는 코드를 추가한다.

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class InitRunner implements CommandLineRunner {

    private final Scheduler scheduler;
    private final RemoteJobClassLoader remoteJobClassLoader;
    private final HelloService helloService;

    @Override
    public void run(String... args) throws Exception {
        log.info("Init Runner executed.");
        JobKey jobKey = JobKey.jobKey("jobkey1", "jobgroup1");
        JobDetail jobDetail = buildJobDetail(jobKey);
        Trigger trigger = buildJobTrigger(jobKey);
        scheduler.scheduleJob(jobDetail, trigger);

        // 여기!!
        try {
            Member dbMember = helloService.saveMember(
                new Member("Homo Efficio", "homo.efficio@gmail.com"));
            log.info("TTT 회원 [{}] 추가됨", dbMember);
        } catch (Exception e) {
            log.error("TTT 회원 추가 중 예외 발생. 메시지: {}",e.getMessage());
        }
    }

    // 이하 생략..
```

실행하고 h2 web console로 확인해보면 롤백되어 레코드가 추가되지 않았음을 확인할 수 있다.

![Imgur](https://i.imgur.com/ulrRfis.png)

`HelloService.saveMember()`에서 `@Transactional`을 제거하면 롤백이 실행되지 않으므로 예외가 발생하더라도 레코드가 추가된 채로 남는다.

![Imgur](https://i.imgur.com/J2aMQsK.png)

이제 작업 클래스 모듈에서도 `@Transactional`이 적용되는지 알아보자.


# 작업 클래스 모듈에서의 `@Transactional` 동작 확인

간단한 확인을 위해 `RemoteSimpleJob` 클래스에서 직접 `MemberRepository`를 통해 `Member`를 저장하는 코드를 작성해보자.

먼저 JPA 리포지토리를 사용할 수 있도록 `build.gradle`에 다음과 같이 spring-data-jpa-starter를 추가해준다.

```groovy
dependencies {
    compile project(':quartz-scheduler')

    compileOnly 'org.projectlombok:lombok:1.18.8'
    annotationProcessor 'org.projectlombok:lombok:1.18.8'
    compile group: 'org.quartz-scheduler', name: 'quartz', version: '2.3.1'
    compile group: 'org.springframework', name: 'spring-context-support', version: '5.1.9.RELEASE'
    compile group: 'org.springframework.data', name: 'spring-data-jpa', version: '2.1.10.RELEASE'  // 여기!!

    testCompile group: 'junit', name: 'junit', version: '4.12'
}

```

## RemoteSimpleJob에서 MemberRepository를 통한 Member 저장

`RemoteSimpleJob`에 `MemberRepository`를 주입하고 `Member`를 저장하는 코드를 추가한다. 아직 `@Transactional`은 추가하지 않은 상태다.

```java
@Slf4j
@RequiredArgsConstructor
public class RemoteSimpleJob implements Job {

    private final HelloService helloService;
    private final MemberRepository memberRepository;  // 여기!!

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("OOO REMOTE JOB [{}] executed.", this.getClass().getSimpleName());
        JobDataMap mergedJobDataMap = context.getMergedJobDataMap();
        mergedJobDataMap.forEach((k, v) -> log.info("OOOOO {}: {}", k, v));

        helloService.sayHello();

        // 여기!!
        try {
            Member dbMember = memberRepository.save(new Member("Homo Efficio", "homo.efficio@gmail.com"));
            log.info("TTT 회원 [{}] 추가됨", dbMember);
            if (1==1) {
                throw new RuntimeException("테스트를 위해 강제로 발생시킨 예외");
            }
        } catch (Exception e) {
            log.error("TTT 회원 추가 중 예외 발생. 메시지: {}",e.getMessage());
        }
    }
}

```

스케줄러 모듈의 `InitRunner`에서 `HelloService.saveMember()` 부분을 다음과 같이 주석처리 한다. 이유는 `Member` 저장을 `RemoteSimpleJob` 내에서 처리하기 때문이다.

```java
@Override
    public void run(String... args) throws Exception {
        log.info("Init Runner executed.");
        JobKey jobKey = JobKey.jobKey("jobkey1", "jobgroup1");
        JobDetail jobDetail = buildJobDetail(jobKey);
        Trigger trigger = buildJobTrigger(jobKey);
        scheduler.scheduleJob(jobDetail, trigger);

        // Member member = new Member("Homo Efficio", "homo.efficio@gmail.com");
        // try {
        //     Member dbMember = helloService.saveMember(member);
        //     log.info("TTT 회원 [{}] 추가됨", dbMember);
        // } catch (Exception e) {
        //     log.error("TTT 회원 추가 중 예외 발생. 메시지: {}",e.getMessage());
        // }
    }
```

실행해보면 롤백이 적용되지 않으므로 다음과 같이 저장 후 예외가 발생하더라도 레코드가 추가되는 것을 확인할 수 있다.

![Imgur](https://i.imgur.com/KOmhtvz.png)

![Imgur](https://i.imgur.com/7cVLJTD.png)

## RemoteSimpleJob에서 @Transactional 사용

이제 다음과 같이 작업 클래스의 `execute()` 메서드에 `@Transactional`을 붙여서 실행해보자.

```java
    @Transactional  // 여기!!
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("OOO REMOTE JOB [{}] executed.", this.getClass().getSimpleName());
        JobDataMap mergedJobDataMap = context.getMergedJobDataMap();
        mergedJobDataMap.forEach((k, v) -> log.info("OOOOO {}: {}", k, v));

        helloService.sayHello();

        try {
            Member dbMember = memberRepository.save(new Member("Homo Efficio", "homo.efficio@gmail.com"));
            log.info("TTT 회원 [{}] 추가됨", dbMember);
            if (1==1) {
                throw new RuntimeException("테스트를 위해 강제로 발생시킨 예외");
            }
        } catch (Exception e) {
            log.error("TTT 회원 추가 중 예외 발생. 메시지: {}",e.getMessage());
        }
    }
```

이번에는 안타깝게도 다음과 같은 에러를 만나게 된다. 지면을 많이 차지하니 [지스트(Gist) 링크](https://gist.github.com/HomoEfficio/b2c9e030b785827b5eba75a1e719387d)로 대신하고, 주요 부분만 살펴보면 다음과 같다.

```
org.quartz.SchedulerException: Job instantiation failed

Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'io.homo_efficio.quartz.job.RemoteSimpleJob': Initialization of bean failed; nested exception is org.springframework.aop.framework.AopConfigException: Could not generate CGLIB subclass of class io.homo_efficio.quartz.job.RemoteSimpleJob: Common causes of this problem include using a final class or a non-visible class; nested exception is org.springframework.cglib.core.CodeGenerationException: java.lang.NoClassDefFoundError-->io/homo_efficio/quartz/job/RemoteSimpleJob

Caused by: org.springframework.aop.framework.AopConfigException: Could not generate CGLIB subclass of class io.homo_efficio.quartz.job.RemoteSimpleJob: Common causes of this problem include using a final class or a non-visible class; nested exception is org.springframework.cglib.core.CodeGenerationException: java.lang.NoClassDefFoundError-->io/homo_efficio/quartz/job/RemoteSimpleJob

Caused by: org.springframework.cglib.core.CodeGenerationException: java.lang.NoClassDefFoundError-->io/homo_efficio/quartz/job/RemoteSimpleJob

Caused by: java.lang.NoClassDefFoundError: io/homo_efficio/quartz/job/RemoteSimpleJob

Caused by: java.lang.ClassNotFoundException: io.homo_efficio.quartz.job.RemoteSimpleJob

```

긴 내용이지만 요약하면 `@Transactional` 기능을 추가하기 위해서 `RemoteSimpleJob`의 프록시 객체를 CGLib 라이브러리를 이용해서 생성해야 하는데, 이때 `RemoteSimpleJob` 클래스를 찾을 수 없다는 얘기다.

`@Transactional`이 없을 때는 프록시 객체를 만들 필요가 없으므로 `RemoteSimpleJob`은 우리가 만든 커스텀 클래스로더를 통해 정상적으로 로딩되어 실행된다. 하지만 **`@Transactional`이 붙어서 CGLib을 통해 프록시 객체를 생성할 때는 우리가 만든 커스텀 클래스로더가 사용되지 못하므로 `RemoteSimpleJob`을 찾지 못하고 위와 같은 에러가 발생하게 된다.** 그림으로 보면 대략 다음과 같다.

![Imgur](https://i.imgur.com/e7GF1s6.png)

그럼 CGLib이 사용하는 클래스로더가 `RemoteSimpleJob`을 로딩할 수 있게 만들면 이 문제도 해결될 것 같다. CGLib는 애플리케이션 구동 환경에서 정해진 클래스로더를 사용하는데 대략 다음과 같다.

- Standalone Tomcat 환경이면 `WebAppClassLoader`
- 스프링부트에 내장된 Embedded Tomcat 환경이면 `LaunchedURLClassLoader`
- 스프링 devtools를 사용하는 로컬 환경이면 `RestartClassLoader`

Standalone Tomcat 환경이라면 `context.xml` 파일에 `<Loader>` 엘리먼트를 통해 커스텀 클래스로더를 지정할 수 있고, 스프링부트 환경이라면 `PropertiesLauncher` 클래스와 `loader.path` 속성으로 클래스로더를 지정할 수 있고, 가장 범용적으로는 manifest 파일에 `Class-Path`로 로딩할 클래스가 포함된 클래스패스를 지정해주면 된다.

이론적으로는 그런데 실제로는 애플리케이션 구동 환경 자체도 로컬 개발 환경, 서버 환경 모두 다르고, 그에 따라 스프링 내부에서 구동되는 CGLib이 `RemoteSimpleJob`을 로딩할 수 있게 하려면 스프링 내부에 대한 심도있는 지식이 필요하다. 그걸로 끝나는 게 아니라 `RemoteSimpleJob`이 참조하는 다른 클래스, 그 클래스가 참조하는 다른 클래스, 그 클래스가 참조하는 다른 클래스... 를 모두 로딩할 수 있어야 한다.

이것저것 시도해보다가 CGLib이 `RemoteSimpleJob`에 대한 프록시 객체를 생성하게 만드는 데 간신히 성공했다. TRACE 로그로 확인할 수 있었다. (이건 예전에 했던 내용이라 클래스 이름 등은 현재 예제와 좀 다르다 ;;)

![Imgur](https://i.imgur.com/Vm9NDZf.png)

그런데 로그를 자세히 보면 `'o.s.aop.framework.CglibAopProxy : Unable to apply any optimizations to advised method: [[[@Transactional_붙어있는_메서드]]]'` 대략 이런 내용이 찍히고, 실제로도 트랜잭션 롤백이 동작하지 않았다.

더 결정적인 문제도 있는데 이렇게 **스프링이 제공해주는 프록시 생성 로직을 통해 프록시로 등록되면 해당 프록시는 캐시된다는 점이다. 그래서 나중에 `RemoteSimpleJob`의 내용을 바꿔서 작업 클래스 모듈의 jar 를 새로 생성한 후에 `RemoteSimpleJob`을 다시 수행해도 새 프록시가 생성되지 않고 기존 내용을 기준으로 생성되어 캐시된 프록시가 사용될 수 있다.** 이러면 작업 클래스 모듈을 분리한 의도가 퇴색되어 버리는 결과가 된다.

이런 모든 문제를 해결하려면 할 수도 있겠지만 나는 이 정도에서 멈추기로 한다. 왜냐하면 **스프링에서는 `@Transactional`이 아니라도 `PlatformTransactionManager`를 사용해서 트랜잭션 기능을 추가할 수 있기 때문이다.**

## PlatformTransactionManager 사용

다음과 같이 `@Transactional`을 제거하고 `PlatformTransactionManager`를 사용하도록 `RemoteSimpleJob`을 개선한다.

```java
@Slf4j
@RequiredArgsConstructor
public class RemoteSimpleJob implements Job {

    private final HelloService helloService;
    private final MemberRepository memberRepository;
    private final PlatformTransactionManager transactionManager;  // 여기!!

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("OOO REMOTE JOB [{}] executed.", this.getClass().getSimpleName());
        JobDataMap mergedJobDataMap = context.getMergedJobDataMap();
        mergedJobDataMap.forEach((k, v) -> log.info("OOOOO {}: {}", k, v));

        helloService.sayHello();

        // 여기!!
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Member dbMember = memberRepository.save(new Member("Homo Efficio", "homo.efficio@gmail.com"));
            log.info("TTT 회원 [{}] 추가됨", dbMember);
            if (1==1) {
                throw new RuntimeException("테스트를 위해 강제로 발생시킨 예외");
            }
            transactionManager.commit(status);  // 여기!!
        } catch (Exception e) {
            log.error("TTT 회원 추가 중 예외 발생. 메시지: {}",e.getMessage());
            transactionManager.rollback(status);  // 여기!!
        }
    }
}
```

실행해보면 다음과 같이 `TTT 회원 [{}] 추가됨` 로그는 찍히지만 그 이후 예외가 발생한다.

![Imgur](https://i.imgur.com/l0xcJB8.png)

이제 레코드가 생성되지 않은 것을 h2 web console에서 확인하면 트랜잭션 처리 문제도 해결된다!

![Imgur](https://i.imgur.com/JDciNBU.png)

오오~ 예상대로 레코드가 생성되지 않았다. 성공!

세 번째 고개인 `@Transactional`은 넘지는 못 했지만, 돌아가는 길을 찾아냈다.


# 정리

>- 스프링부트 애플리케이션에서 분리되어 나간 작업 클래스 모듈에서는 `@Transactional`을 일반적인 경우처럼 쉽게 사용할 수 없다.
>
>- 이유는 `@Transactional` 기능을 추가하기 위해서 `RemoteSimpleJob`의 프록시 객체를 CGLib 라이브러리를 이용해서 생성해야 하는데, 이때 `RemoteSimpleJob` 클래스를 찾을 수 없기 때문이다.
>
>- 작업 클래스 모듈에서 `@Transactional`은 사용할 수 없지만 `PlatformTransactionManager`를 이용하면 트랜잭션 처리를 할 수 있다.

# 시리즈 마무리

총 3편에 걸쳐 Quartz 적용 아키텍처를 개선하는 과정을 살펴봤다.

이를 통해 **변경 주기가 다른 스케줄러와 작업 클래스를 별도의 모듈로 분리하고, 분리된 모듈에 의존 관계를 주입해서 트랜잭션까지 처리할 수 있는 클린 Quartz 아키텍처를 적용할 수 있게 됐다.**

## 1편 모듈 분리

[1편](https://homoefficio.github.io/2019/09/28/Quartz-스케줄러-적용-아키텍처-개선-1/)의 키워드는 **모듈 분리와  `URLClassLoader`**

![Imgur](https://i.imgur.com/5RHsMzy.png)


## 2편 의존 관계 주입

[2편](https://homoefficio.github.io/2019/09/29/Quartz-스케줄러-적용-아키텍처-개선-2/)의 키워드는 **의존 관계 주입과 `SpringBeanJobFactory`**

![Imgur](https://i.imgur.com/mT6CfNb.png)


## 3편 트랜잭션 처리

[3편](https://homoefficio.github.io/2019/09/29/Quartz-스케줄러-적용-아키텍처-개선-3/)의 키워드는 **트랜잭션 처리와 `PlatformTransactionManager`**

![Imgur](https://i.imgur.com/dbHHXDv.png)


