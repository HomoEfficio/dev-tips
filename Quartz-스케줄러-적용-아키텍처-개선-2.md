# Quartz 스케줄러 적용 아키텍처 개선 - 2

[1편](https://homoefficio.github.io/2019/09/28/Quartz-스케줄러-적용-아키텍처-개선-1/)에서는 Quartz 스케줄러 적용 시 변경 주기가 다른 스케줄러 모듈과 작업 클래스 모듈을 분리해서 클린 아키텍처에 다가가는 방법을 알아봤다.

분리된 작업 클래스 모듈은 DB 작업을 할 수도 있고, 하둡 인프라 관련 작업을 할 수도 있고, 알림 메일도 보내야하는 등 여러 작업을 할 수 있어야 한다. 그런데 작업 클래스 모듈은 말 그대로 작업 클래스만 모아 놓은 jar 라이브러리일 뿐이라서 JDBC 드라이버나 SMTP 메일 서버 설정 등을 포함하고 있지 않으며, 이런 것들을 스스로 가지고 있는 것 자체도 Single Responsibility Principle 관점에서 보면 적절하지 않다.

그래서 작업 클래스 모듈을 동적으로 로딩하는 스케줄러 모듈이 작업 클래스가 필요로 하는 컴포넌트를 주입해주는 구조로 구성할 수 있다면 가장 좋다. 즉, 다음과 같이 `@Autowire`를 통해 작업 클래스에 필요한 의존 관계를 주입해 줄 수 있다면 딱 좋다. 

![Imgur](https://i.imgur.com/mT6CfNb.png)

그런데 보통 `@Autowire`는 스프링 애플리케이션이 구동되면서 bean을 생성하고 객체 협력망을 구성할 때 작동한다. 지금처럼 스케줄러를 포함한 스프링 애플리케이션이 완전히 구동된 후에 동적으로 작업 클래스를 로딩할 때도 `@Autowire`를 통해 의존 관계 주입하는 것이 가능할까?


# SpringBeanJobFactory

결론부터 말하면 **스프링부트가 제공하는 `SpringBeanJobFactory`를 통해 애플리케이션 구동 완료 후에 동적으로 추가하는 bean에도 의존 관계를 쉽게 주입할 수 있다.**

예제에서는 편의상 스케줄링 직후 작업이 수행되도록 작성했지만, 스케줄러는 일반적으로 작업을 스케줄하는 시점과 작업을 실행하는 시점이 다르다. Quartz 스케줄러도 마찬가지며 스케줄하는 시점에도 작업 클래스를 로딩하지만 작업을 실행하는 시점에도 클래스를 로딩한다. 그리고 작업을 실행하려면 작업 클래스를 인스턴스화 해야 하는데, 이 때 `JobFactory`가 사용된다. [문서](https://www.quartz-scheduler.org/api/2.3.1-SNAPSHOT/index.html)에도 나와있는 것처럼 **`JobFactory`를 의존 관계 주입 통로로 사용할 수 있다.**

스프링은 꽤 오래 전부터 Quartz를 지원해오고 있으며, 스프링부트에도 quartz starter가 있고 그 안에 `JobFactory`를 구현한 [`SpringBeanJobFactory`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/quartz/SpringBeanJobFactory.html)가 있다. 애플리케이션 실행 로그를 살펴보면 `SpringBeanJobFactory`가 사용되고 있음을 알 수 있다.

![Imgur](https://i.imgur.com/6cElvr0.png)

따라서 **스프링부트를 통해 quartz를 사용하고 있다면, `SpringBeanJobFactory` 덕분에 작업 클래스도 일반적인 스프링 bean과 마찬가지로 간단하게 의존 관계 주입이 가능하다.** 이제 관련 코드를 작성하면서 알아보자.

## 주입할 컴포넌트 생성

먼저 주입할 컴포넌트를 만들자. 스케줄러 모듈 쪽에 다음과 같이 단순한 `HelloService`를 추가한다.

```java
@Service
@Slf4j
public class HelloService {

    public void sayHello() {
        log.info("OOO {}.sayHello() executed", this.getClass().getSimpleName());
    }
}
```

## 모듈 의존 관계 추가

작업 클래스 모듈에 있는 `RemoteSimpleJob`가 스케줄러 모듈에 있는 `HelloService`를 주입 받으려면 다음과 같이 코드 상으로 `HelloService`가 명시돼야 하는데, 작업 클래스 모듈에서는 `HelloService`를 인식할 수 없으며, 컴파일 에러가 발생한다.

따라서 작업 클래스 모듈의 `build.gradle` 파일에 다음과 같이 스케줄러 모듈(quartz-scheduler)에 대한 의존 관계를 추가한다.

```groovy
dependencies {
    compile project(':quartz-scheduler')  // 여기!!

    compileOnly 'org.projectlombok:lombok:1.18.8'
    annotationProcessor 'org.projectlombok:lombok:1.18.8'
    compile group: 'org.quartz-scheduler', name: 'quartz', version: '2.3.1'
    compile group: 'org.springframework', name: 'spring-context-support', version: '5.1.9.RELEASE'

    testCompile group: 'junit', name: 'junit', version: '4.12'
}
```

## 작업 클래스에 의존 관계 추가

다음과 같이 스케줄러 모듈에 있는 `HelloService`를 주입 받아서 사용하는 코드를 `RemoteSimpleJob`에 추가한다.

```java
@Slf4j
@RequiredArgsConstructor  // 여기!!
public class RemoteSimpleJob implements Job {

    private final HelloService helloService;  // 여기!!


    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("OOO REMOTE JOB [{}] executed.", this.getClass().getSimpleName());
        JobDataMap mergedJobDataMap = context.getMergedJobDataMap();
        mergedJobDataMap.forEach((k, v) -> log.info("OOOOO {}: {}", k, v));

        helloService.sayHello();  // 여기!!
    }
}
```

작업 클래스 모듈에 추가해둔 `deployJar`를 실행하고 애플리케이션을 다시 실행하면 `RemoteSimpleJob`이 `HelloService`를 주입 받아서 사용함을 확인할 수 있다.

![Imgur](https://i.imgur.com/wBqMrZP.png)

자 이제 의존 관계 주입이라는 두 번째 고개를 넘었다. 그래서 DB 작업도 할 수 있게 됐다. 그런데 DB 작업을 할 때는 트랜잭션 처리를 해줘야 한다. 스프링에서는 `@Transactional`을 통해 아주 쉽게 트랜잭션 처리를 할 수 있다. 작업 클래스에서도 `@Transactional`을 사용할 수 있을까? 마지막 세 번째 고개는 `@Transactional`이다. 3편에서
다룬다.


# 정리

>- Quartz 작업 클래스 모듈이 실무에서 발생하는 다양한 작업을 수행하려면 관련 컴포넌트를 주입 받아야 한다.
>
>- 스프링부트에서 제공하는 `SpringBeanJobFactory` 덕분에 Quartz 작업 클래스에서도 일반적인 bean과 마찬가지 방식으로 간단하게 의존 관계를 주입 받을 수 있다.


# (번외편) 스프링부트가 아닌 그냥 스프링일 경우

스프링부트가 아니라 그냥 스프링에서 Quartz를 사용할 때도 의존 관계 주입이 가능하다.

Quartz에서는 `JobFactory`라는 의존 관계 주입 통로를 제공하고 있고, 스프링에서는 `JobFactory`의 구현체인 `AdaptableJobFactory`를 스프링 2.0부터 제공해오고 있다. 그리고 스프링에는 의존 관계를 주입하면서 bean을 만들어 주는 `AutowireCapableBeanFactory`가 있다. 따라서 다음과 같은 커스텀 팩토리를 만들어주고, 이를 Quartz 스케줄러에서 사용할 수 있게 설정해주면 된다.

## 커스텀 작업 팩토리

커스텀 작업 팩토리의 역할은 작업 클래스 인스턴스 생성과 의존 관계 주입하기, 이렇게 두 가지다.

작업 클래스 인스턴스 생성은 `AdaptableJobFactory`에 이미 있는 기능이다. 여기에 의존 관계 주입 기능만 추가해주면 되므로 `AdaptableJobFactory`를 상속받아서 해당 부분만 오버라이드 하면 된다.

의존 관계를 주입하려면 `AutowireCapableBeanFactory`가 필요하고, `AutowireCapableBeanFactory`를 얻으려면 `ApplicationContext`가 필요하다. 어떻게든 `ApplicationContext`를 주입 받을 수 있으면 그만이지만 딱 이런 역할을 의미하는 `ApplicationContextAware`라는 인터페이스가 있으니 구현해주자.

```java
public class QuartzJobFactory extends AdaptableJobFactory implements ApplicationContextAware {

    private ApplicationContext applicationContext;


    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
        Object jobInstance = super.createJobInstance(bundle);
        autowireBean(jobInstance);
        return jobInstance;
    }

    private void autowireBean(Object jobInstance) {
        AutowireCapableBeanFactory autowireCapableBeanFactory = applicationContext.getAutowireCapableBeanFactory();
        autowireCapableBeanFactory.autowireBean(jobInstance);
    }
}
```

## 커스텀 스케줄러 팩토리 및 @Configuration 클래스

스프링부트는 starter를 통해 자동 설정한 Quartz 스케줄러를 제공하지만 스프링이라면 스케줄러도 직접 생성하고 설정해줘야 한다. 그리고 앞에서 만든 커스텀 작업 팩토리를 직접 생성한 스케줄러에 설정해주면 된다.

스프링에서는 4.1부터 Quartz 스케줄러 생성 작업을 쉽게 해주는 `SchedulerFactoryBean`도 제공한다. Quartz 2.1.4 이상이면 호환 가능하다. 따라서 다음과 같이 커스텀 스케줄러 팩토리를 만들어서 Autowiring 기능이 탑재된 커스텀 작업 팩토리를 설정해주고, Quartz 스케줄러를 bean으로 반환하는 설정 클래스를 작성하면 된다.

```java
@Configuration
public class QuartzConfig {

    @Bean
    public Scheduler scheduler(ApplicationContext applicationContext) {
        SchedulerFactoryBean schedulerFactoryBean = getSchedulerFactoryBean(applicationContext);
        return schedulerFactoryBean.getScheduler();
    }
    
    private SchedulerFactoryBean getSchedulerFactoryBean(ApplicationContext applicationContext) {
        QuartzJobFactory quartzJobFactory = new QuartzJobFactory();
        quartzJobFactory.setApplicationContext(applicationContext);

        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        schedulerFactoryBean.setJobFactory(quartzJobFactory);
//        schedulerFactoryBean.setXXX(...); // 기타 스케줄러 설정값 직접 지정 필요

        return schedulerFactoryBean;
    }
}
```
