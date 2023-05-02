## Given, When

다음과 같이 `@OneToMany`, `@ManyToMany`, `@ElementCollection`으로 `1:N` 또는 `M:N`의 관계일 때, `FetchType.EAGER`로 List를 가져오면, 

```java
@Entity
@Table(name = "BATCH_TASK")
@Getter
public class BatchTask {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "batch_task_id")
    private Long id;

    private String name;

    @ElementCollection(fetch = FetchType.EAGER)
    private List<Long> parentTaskIds = new ArrayList<>();

    @ElementCollection(fetch = FetchType.EAGER)
    private List<Long> childTaskIds = new ArrayList<>();

    // 이하 생략
}
```

## Then

다음과 같이 `MultipleBagFetchException`이 발생한다. 

이 예외는 Hibernate 5.0.12(Spring Boot 1.5.16)과 Hibernate 5.2.17(Spring Boot 2.0.5)에서 모두 발생한다.

```
2018-10-09 16:57:48.969 ERROR 13140 --- [  restartedMain] o.s.boot.SpringApplication               : Application startup failed

org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'entityManagerFactory' defined in class path resource [org/springframework/boot/autoconfigure/orm/jpa/HibernateJpaAutoConfiguration.class]: Invocation of init method failed; nested exception is javax.persistence.PersistenceException: [PersistenceUnit: default] Unable to build Hibernate SessionFactory; nested exception is org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags: [io.homo.efficio.experimental.jpa.domain.BatchTask.childTaskIds, io.homo.efficio.experimental.jpa.domain.BatchTask.parentTaskIds]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1631) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:553) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:481) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:312) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:230) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:308) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:197) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:1080) ~[spring-context-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:857) ~[spring-context-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:543) ~[spring-context-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.refresh(EmbeddedWebApplicationContext.java:122) ~[spring-boot-1.5.16.RELEASE.jar:1.5.16.RELEASE]
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:693) [spring-boot-1.5.16.RELEASE.jar:1.5.16.RELEASE]
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:360) [spring-boot-1.5.16.RELEASE.jar:1.5.16.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:303) [spring-boot-1.5.16.RELEASE.jar:1.5.16.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1118) [spring-boot-1.5.16.RELEASE.jar:1.5.16.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1107) [spring-boot-1.5.16.RELEASE.jar:1.5.16.RELEASE]
	at io.homo.efficio.experimental.jpa.SpringBootJpaExperimentalApplication.main(SpringBootJpaExperimentalApplication.java:10) [classes/:na]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_181]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_181]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_181]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_181]
	at org.springframework.boot.devtools.restart.RestartLauncher.run(RestartLauncher.java:49) [spring-boot-devtools-1.5.16.RELEASE.jar:1.5.16.RELEASE]
Caused by: javax.persistence.PersistenceException: [PersistenceUnit: default] Unable to build Hibernate SessionFactory; nested exception is org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags: [io.homo.efficio.experimental.jpa.domain.BatchTask.childTaskIds, io.homo.efficio.experimental.jpa.domain.BatchTask.parentTaskIds]
	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.buildNativeEntityManagerFactory(AbstractEntityManagerFactoryBean.java:396) ~[spring-orm-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.afterPropertiesSet(AbstractEntityManagerFactoryBean.java:371) ~[spring-orm-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean.afterPropertiesSet(LocalContainerEntityManagerFactoryBean.java:336) ~[spring-orm-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1689) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1627) ~[spring-beans-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	... 21 common frames omitted
Caused by: org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags: [io.homo.efficio.experimental.jpa.domain.BatchTask.childTaskIds, io.homo.efficio.experimental.jpa.domain.BatchTask.parentTaskIds]
	at org.hibernate.loader.plan.exec.internal.AbstractLoadQueryDetails.generate(AbstractLoadQueryDetails.java:178) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.plan.exec.internal.EntityLoadQueryDetails.<init>(EntityLoadQueryDetails.java:89) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.plan.exec.internal.BatchingLoadQueryDetailsFactory.makeEntityLoadQueryDetails(BatchingLoadQueryDetailsFactory.java:61) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.entity.plan.AbstractLoadPlanBasedEntityLoader.<init>(AbstractLoadPlanBasedEntityLoader.java:82) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.entity.plan.EntityLoader.<init>(EntityLoader.java:103) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.entity.plan.EntityLoader.<init>(EntityLoader.java:38) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.entity.plan.EntityLoader$Builder.byUniqueKey(EntityLoader.java:83) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.entity.plan.EntityLoader$Builder.byPrimaryKey(EntityLoader.java:77) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.entity.plan.AbstractBatchingEntityLoaderBuilder.buildNonBatchingLoader(AbstractBatchingEntityLoaderBuilder.java:30) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.loader.entity.BatchingEntityLoaderBuilder.buildLoader(BatchingEntityLoaderBuilder.java:59) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.persister.entity.AbstractEntityPersister.createEntityLoader(AbstractEntityPersister.java:2306) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.persister.entity.AbstractEntityPersister.createEntityLoader(AbstractEntityPersister.java:2328) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.persister.entity.AbstractEntityPersister.createLoaders(AbstractEntityPersister.java:3928) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.persister.entity.AbstractEntityPersister.postInstantiate(AbstractEntityPersister.java:3910) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.internal.SessionFactoryImpl.<init>(SessionFactoryImpl.java:446) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.boot.internal.SessionFactoryBuilderImpl.build(SessionFactoryBuilderImpl.java:444) ~[hibernate-core-5.0.12.Final.jar:5.0.12.Final]
	at org.hibernate.jpa.boot.internal.EntityManagerFactoryBuilderImpl.build(EntityManagerFactoryBuilderImpl.java:879) ~[hibernate-entitymanager-5.0.12.Final.jar:5.0.12.Final]
	at org.springframework.orm.jpa.vendor.SpringHibernateJpaPersistenceProvider.createContainerEntityManagerFactory(SpringHibernateJpaPersistenceProvider.java:60) ~[spring-orm-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean.createNativeEntityManagerFactory(LocalContainerEntityManagerFactoryBean.java:360) ~[spring-orm-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.buildNativeEntityManagerFactory(AbstractEntityManagerFactoryBean.java:384) ~[spring-orm-4.3.19.RELEASE.jar:4.3.19.RELEASE]
	... 25 common frames omitted


Process finished with exit code 0
```

## 원인

에러 메시지를 보면 다음과 같이 **동시에 다수의 `bag`를 사용할 수 없다**라고 나온다. List를 사용하면 Hibernate에서는 `PersistentBag`에 데이터를 담아 가져오는데,

>Caused by: org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags

왜 동시에 다수의 `bag`를 사용할 수 없는지는 찾아봐도 잘 없어서 모르겠다.


## 해결

세 가지 해법이 있다.

### Lazy로 가져오기

Lazy로 가져오면 `FETCH JOIN`을 하지 않고 2번의 쿼리를 통해 가져오므로 동시에 bag을 사용하지 않게 되어 에러가 발생하지 않는 것 같다.

```java

    @ElementCollection(fetch = FetchType.LAZY)
    private List<Long> parentTaskIds = new ArrayList<>();

    @ElementCollection(fetch = FetchType.LAZY)
    private List<Long> childTaskIds = new ArrayList<>();
    
```

![Imgur](https://i.imgur.com/yCzfGOV.png)

### Set 사용

또는 Eager로 가져오되 다음과 같이 List 대신 Set을 사용하면 된다. Set을 사용하면 `PersistentBag`이 아니라 `PersistentSet`에 담아 가져오며, `PersistentSet`에는 동시에 담아 가져올 수 있는 모양이다.

```java

    @ElementCollection(fetch = FetchType.EAGER)
    private Set<Long> parentTaskIds = new HashSet<>();

    @ElementCollection(fetch = FetchType.EAGER)
    private Set<Long> childTaskIds = new HashSet<>();
    
```

![Imgur](https://i.imgur.com/nHFDIUO.png)

### Hibernate의 `@LazyCollection(LazyCollectionOption.FALSE)` 사용

다음과 같이 Hibernate에서 제공하는 `@LazyCollection(LazyCollectionOption.FALSE)`를 사용해도 에러 없이 동시에 2개의 List 값을 가져올 수 있다.

```java
    @ElementCollection(fetch = FetchType.EAGER)
    private List<Long> parentTaskIds = new ArrayList<>();

    @ElementCollection(fetch = FetchType.EAGER)
    @LazyCollection(LazyCollectionOption.FALSE)    
    private List<Long> childTaskIds = new ArrayList<>();
```

## 정리

JPA(Hibernate)에서 `List`를 `FetchType.EAGER`로 가져오면 `MultipleBagFetchException`이 발생한다. 이를 우회하려면,

> `FetchType.LAZY`로 가져오거나,
>
> `List` 대신 `Set`을 사용하거나,
>
> `@LazyCollection(LazyCollectionOption.FALSE)`를 사용한다.

