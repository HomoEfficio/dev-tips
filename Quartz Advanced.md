# Quartz Advanced

# Quartz를 사용하는 이유

- Clustering
- 트리거를 통한 Job 상태 관리
    - 스케줄링 정보 유지
    - misfire 처리
    
# Quartz 단점

- 클러스터의 로드 분산이 단순히 Round-robin 이라서 실질적인 부하 분산이 안 될 때도 있음
- 클러스터 내 특정 노드에 과부하가 걸리면 해당 Job은 실제로 계속 실행되더라도 해당 Job의 Quartz DB에서 Trigger는 종료된 것으로 나옴
- 특정 Job의 강제 Kill이 항상 동작하지는 않음

# Spring Boot 통합

- Job 클래스에서 인스턴스를 생성해서 스프링의 Bean으로 등록해주는 SpringBeanJobFactory를 구현하는 클래스 필요
- Job을 스케줄링에 등록하면 Job의 실제 구현 클래스 정보가 Quartz에 등록됨
- 스케줄러에 의해서 정해진 시간에 Job 이 실행될 때 이미 등록되어 있던 클래스 정보를 이용해서 클래스를 로딩하고 스프링 빈으로 등록
  - 따라서 Job 구현 클래스도 스프링을 통해 필요한 의존 관계를 주입받을 수 있다.

## QuartzConfig

```java
@Configuration
@Import({QuartzProperties.class, DMPTriggerListener.class})
public class QuartzConfig {
    @Autowired
    ApplicationContext applicationContext;

    @Autowired
    QuartzProperties quartzProperties;

    @Autowired
    TriggerListener myTriggerListener;

    @Value("${quartz.jobRepo}")
    private String quartzJobRepo;

    @Bean
    public JobFactory jobFactory(ApplicationContext applicationContext) {
        QuartzJobFactory quartzJobFactory = new QuartzJobFactory();
        quartzJobFactory.setApplicationContext(applicationContext);
        return quartzJobFactory;
    }

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean(ApplicationContext applicationContext) {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setOverwriteExistingJobs(true);
        factory.setJobFactory(jobFactory(applicationContext));
        factory.setQuartzProperties(quartzProperties.toProperties());
        return factory;
    }

    @Bean
    public Scheduler scheduler() throws SchedulerException {
        Scheduler scheduler = schedulerFactoryBean(applicationContext).getScheduler();
        scheduler.getListenerManager().addTriggerListener(myTriggerListener);
        scheduler.start();
        return scheduler;
    }

    public class QuartzJobFactory extends SpringBeanJobFactory implements ApplicationContextAware {

        private transient AutowireCapableBeanFactory beanFactory;

        @Override
        public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
            beanFactory = applicationContext.getAutowireCapableBeanFactory();
        }

        @Override
        protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
            final Object job = super.createJobInstance(bundle);
            beanFactory.autowireBean(job);
            return job;
        }
    }
}
```

## TriggerListener

Trigger는 Job의 실행 주기와 실행 상태를 관리한다. `TriggerListener`는 Job이 시작, 종료될 때 Quartz에 의해 자동으로 호출된다. 상태관리나 메일 발송 등을 Trigger에 담을 수 있다. `@Component`를 붙여서 빈으로 등록해줘야, `QuartzJobFactory`를 생성할 때 스케줄러에 리스너로 등록할 수 있다.

```java
@Component
public class MyTriggerListener implements TriggerListener {

    @Autowired
    ...
    
    @Override
    public void triggerFired(Trigger trigger, JobExecutionContext context) {
    }
    
    @Override
    public void vetoJobExecution(Trigger trigger, JobExecutionContext context) {
    }
    
    @Override
    public void triggerMisfired(Trigger trigger) {
    }
    
    @Override
    public void triggerComplete(Trigger trigger, JobExecutionContext context, Trigger.CompletedExecutionInstruction triggerInstructionCode) {
    }
}

```


# Job 클래스 외부화

Quartz 스케줄러가 실행되는 스프링 부트 애플리케이션(이후 그냥 Quartz 애플리케이션이라 한다)에 Job 클래스도 포함시키면, Job을 추가/변경/삭제할 때마다 Quartz 애플리케이션을 재배포해야 한다.

일반적으로 Quartz 애플리케이션에는 스케줄에 따라 여러 Job 들이 실행되고 있으므로, 실행되는 Job 이 없을 때 배포해야 하는데 이는 운영에 매우 큰 제약사항이 된다. 따라서 Quartz 애플리케이션에는 Job과 관련된 비즈니스 데이터의 관리, 스케줄러 자체에 대한 내용 만을 포함하고, 실제 Job 클래스는 외부화하는 것이 좋다..를 넘어서 필수적이다.

Job을 외부화해도 Job이 실행될 때 Quartz 애플리케이션의 빈으로 등록이 되므로 필요한 의존 관계를 주입 받아서 필요한 작업을 수행할 수 있다. 다만 한 가지 극복할 수는 있지만 작지 않은 제약사항이 있는데, 동적으로 로딩되어 Bean으로 등록되는 Job 클래스에는 스프링 부트 애플리케이션이 기동될 때 정적으로 적용되는 `@Transactional`을 사용할 수 없다는 점이다.

- CustomClassLoader 설정
- CustomClassLoader 구현

# 예외 처리

- 강제 종료: 절대 Job을 구현하지 말고 반드시 InterruptableJob 을 구현해서 강제 종료 가능하게 해야함
  - Trigger.triggerComplete () 내부에서 예외 발생 시 해당 trigger가 계속 `BLOCKED` 상태로 남아있는 오류 있음
  - InterruptableJob이 구현된 Job만 강제로 Kill 해서 `BLOCKED` 상태 해제 가능
  - 기타 여러 이유로 `BLOCKED` 상태로 남아있는 trigger를 Kill 하기 위해 반드시 필요
- 비정상 예외

# 이상 동작

- 클러스터 노드 중 1대가 불량이 되면 불량 인스턴스에서 실행 중이던 Trigger 들이 fired_triggers 테이블에서 사라진다.
  - 사라진 후에도 실제로는 계속 실행되기도 하며
  - 불량 노드에서 실행 중인 Job은 자동 복구 및 재실행이 되지 않도록 설정(non-recovery)하더라도 다른 노드에 의해 Job이 복구되어 자동으로 재실행되기도 한다.
- Trigger.triggerComplete() 내부에서 예외 발생 시 해당 trigger가 계속 `BLOCKED` 상태로 남아있는 오류 있음

# 워크플로우

- 스케줄링 병렬화와 실행 스레드의 얽힘  

