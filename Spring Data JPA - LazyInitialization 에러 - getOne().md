# Spring Data JPA - LazyInitialization 에러

스프링 데이터 JPA를 사용하다보면 가끔 만나는 정겨운 에러가 있다.

```
LazyInitializationException: could not initialize proxy - no session
```

요 에러는 말그대로 지연 로딩을 하려는데 이미 세션이 사라져서 지연 로딩을 못할 때 발생하는 에러다.

그래서 보통은 `@*ToMany` 관계로 되어 있어 기본 FetchType이 Lazy인 멤버를 읽어올 때 발생하며, 그냥 쉬운 해결 법으로는 `FetchType.EAGER`를 명시해서 즉시 로딩으로 읽어오게 하면 해결될 수도 있다. 

하지만, 비즈니스 요구가 아니라 단순히 LazyInitialization 에러를 피하기 위해 즉시 로딩을 선택하는 것은 좋은 해법이 아니다.

요는 지연 조회 시점까지 세션을 유지 시켜주면 되는데 **가장 쉽게는 해당 조회가 포함된 메서드에 `@Transactional`(조회일 때는 `@Transactional(readOnly = true)`)를 붙여주면 된다.**

그런데 이렇게 해도 계속 LazyInitialization 에러가 발생하는 경우를 만났다!!

나중에 알고보니 꽤 허탈했는데 원인은 `xxxRepository.findOne(id)`를 호출해야되는데 `xxxRepository.getOne(id)`를 호출했기 때문이었다.

아마도 오타로 무심결에 f 대신 g를 입력했는데 IDE가 자동 완성으로 `getOne`을 추천해주고, 습관적으로 그냥 추천해준대로 선택해서 getOne()이 묻어간 듯 하다.

IDE 자동 완성 기능에도 병폐가 있다는 사실을 알게 되었다..


## 서비스 클래스의 메서드가 아닌 메서드에서 LazyCollection 호출 시

상황에 따라 서비스 클래스의 메서드가 아닌 메서드에서 LazyCollection을 호출해야할 때가 있다.

### 1. 조회 부분을 서비스 메서드 내로 이동

내 경우 Quartz Job 클래스에서 `getTaskList()` 같은 형태로 Task 목록을 불러와야 하는데, Quartz Job 클래스는 서비스 클래스가 아니며, 유일한 메서드인 `execute()`에 `@Transactional`을 붙이는 것도 부적합하다. 이런 상황에서 Collection을 읽어오기 위해 위해 `FetchType.EAGER`를 쓰면 Lazy를 써도 되는 곳에서조차 EAGER로 동작하므로 좋지 않다. 어떻게 하면 좋을까?

해법은 단순하다.

서비스 클래스 내에 다음과 같이 `Hibernate.initialize()`를 이용해서 EAGER 용 메서드를 만들고, Job 클래스에서 이 메서드를 호출해서 쓰면 된다.

```java
    @Transactional(readOnly = true)
    public List<Task> getTaskListEager(Long tasksId) {
        List<Task> taskList = this.tasksRepository.findOne(tasksId).getTaskList();
        Hibernate.initialize(taskList);
        return taskList;
    }
```

이 방식에도 주의할 점이 있는데, `Hibernate.initialize(taskList)`가 `taskList` 내에 중첩되어 있는 Collection 까지 가져오지는 못한다는 점이다.

LazyCollection 내에 중첩된 LazyCollection 가 또 있을 때는 그 중첩된 LazyCollection 에 대해서도 `Hibernate.initialize()`를 다시 해줘야 한다.

### 2. @Transactional 대신 TransactionManager 사용

굳이 `@Transactional`을 고집할 필요가 없다.

다음과 같이 Spring에서 제공하는 `PlatformTransactionManager`를 사용해서 직접 session 을 만들어주면 된다.

```java
    
    ...
    @Autowired
    private PlatformTransactionManager transactionManager;
    ...
    
    public List<Task> getTaskListEager(Long tasksId) {
        TransactionStatus transactionStatus = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        List<Task> taskList = null;
        try {
            taskList = this.tasksRepository.findOne(tasksId).getTaskList();
            this.transactionManager.commit(transactionStatus);
        } catch (Exception e) {
            this.transactionManager.rollback(transactionStatus);
        }
                
        return taskList;
    }
    
```


