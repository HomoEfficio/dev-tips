# Lombok `@RequiredArgsConstructor(onConstructor = @__(@Autowired))` 주의 사항

스프링에서 생성자 주입을 사용할 때 다음과 같이 Lombok의 애노테이션을 쓰면, 생성자에 주입할 요소가 추가되더라도 코드를 손 댈 필요가 없으므로 편리하다.

```java
@Component
@Aspect
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class QueryAspect {

    @NonNull
    private AsyncQueryResultRepository repository;

    @NonNull
    private ThreadPoolTaskExecutor taskExecutor;
    
    // 생성자를 명시적으로 작성하지 않아도 Lombok이 나중에 만들어 줌    
```

그런데 위 방식이 안 먹을 때가 있다. 아래와 같이 `@Qualifier`를 붙여 주입받아야 할 때 위 Lombok 방식을 쓰면,

```java
@Component
@Aspect
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class QueryAspect {

    @NonNull
    private AsyncQueryResultRepository repository;

    @NonNull
    @Qualifier("logToDBThreadPoolTaskExecutor")
    private ThreadPoolTaskExecutor taskExecutor;
```

다음과 같은 에러가 발생한다.

```
***************************
APPLICATION FAILED TO START
***************************

Description:

Parameter 1 of constructor in x.x.x.aop.querying.QueryAspect required a single bean, but 2 were found:
	- longRunningThreadPoolTaskExecutor: defined by method 'longRunningThreadPoolTaskExecutor' in class path resource [.../ThreadPoolTaskExecutorConfig.class]
	- logToDBThreadPoolTaskExecutor: defined by method 'logToDBThreadPoolTaskExecutor' in class path resource [.../ThreadPoolTaskExecutorConfig.class]


Action:

Consider marking one of the beans as @Primary, updating the consumer to accept multiple beans, or using @Qualifier to identify the bean that should be consumed
```

아무래도 롬복이 만들어주는 생성자에서는 `@Qualifier`가 제대로 인식되지 못하는 것 같다. 따라서 `@Qualifier`를 붙여 주입받아야 할 때는 아래와 같이 Lombok 대신 명시적으로 생성자를 작성해줘야 한다.

```java
@Component
@Aspect
public class QueryAspect {

    private AsyncQueryResultRepository repository;

    private ThreadPoolTaskExecutor taskExecutor;

    public QueryAspect(@NonNull AsyncQueryResultRepository repository,
                       @NonNull @Qualifier("logToDBThreadPoolTaskExecutor") ThreadPoolTaskExecutor taskExecutor) {
        this.repository = repository;
        this.taskExecutor = taskExecutor;
    }
```
