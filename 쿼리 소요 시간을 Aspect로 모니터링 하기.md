# 쿼리 소요 시간을 Aspect로 모니터링 하기

Impala를 사용해서 대용량 데이터를 대상으로 실시간 쿼리 기능을 사용하고 있는데, 노드 현황에 따라 동일한 쿼리임에도 소요시간이 오래 걸리는 경우가 있다.

모니터링을 위해 Aspect를 사용해서 Impala 쿼리 메서드에 `Around` 어드바이스를 적용하고, 수행 쿼리, 소요 시간 등의 모니터링 정보를 MySQL에 저장하고자 한다.

## 구현 코드

대략 다음과 같이 구현하면 될 것 같다.

```java
@Component
@Aspect
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class QueryAspect {

    @NonNull
    private AsyncQueryResultRepository repository;

    @Around("execution(* a.b.c.d.e.f.g.ImpalaQueryExecutor.query*(..))")
    public Object onAroundImpalaQuery(ProceedingJoinPoint joinPoint) {

        long startTime = System.currentTimeMillis();
        Object[] args = joinPoint.getArgs();
        String queryTemplate = (String) args[0];
        Object[] queryParams = args.length == 2 ? (Object[]) args[1] : new Object[] {};
        String mappedQuery = getMappedQuery(queryTemplate, queryParams);
        Object result = null;

        try {
            Object queryResult = joinPoint.proceed();
            final long endTime = System.currentTimeMillis();
            result = queryResult;
            this.repository.save(
                    new AsyncQueryResult(
                            mappedQuery,
                            queryResult instanceof List ? ((List) queryResult).size() : 1,
                            new Date(startTime),
                            new Date(endTime),
                            (endTime - startTime) / 1000.0f,
                            "OK"
                    )
            );
        } catch (Throwable throwable) {
            final long endTime = System.currentTimeMillis();            
            this.repository.save(
                    new AsyncQueryResult(
                            mappedQuery,
                            -1,
                            new Date(startTime),
                            new Date(endTime),
                            (endTime - startTime) / 1000.0f,
                            "FAIL"
                    )
            );
            // 원래 메서드에서 처리하게 함
            throw new RuntimeException(throwable);
        }
        return result;
    }

    private String getMappedQuery(String queryTemplate, Object[] queryParams) {
        String[] splitted = queryTemplate.split("\\?");

        StringBuilder sb = new StringBuilder();
        for (int i = 0, len = queryParams.length; i < len; i++) {
            sb.append(splitted[i]).append(getParamString(queryParams[i]));
        }
        if (splitted.length > queryParams.length) {
            sb.append(splitted[splitted.length - 1]);
        }

        return sb.toString();
    }

    private String getParamString(Object param) {
        String result = "";
        if (param instanceof String) {
            result = "'" + param + "'";
        } else if (param instanceof Integer
                || param instanceof Long
                || param instanceof Float
                || param instanceof Double) {
            result = String.valueOf(param);
        }
        return result;
    }
}
```

그런데 이렇게 구현하면 다음과 같은 오류가 발생할 수 있다.

>Caused by: java.sql.SQLException: Connection is read-only. Queries leading to data modification are not allowed

읽기 전용 연결에서 수정은 허용되지 않는다는 얘기다.

`@Transaction(readOnly = true)`가 붙어있는 메서드에서 Impala 쿼리를 실행하는데, Aspect에서 MySQL에 쓰기를 발생시키므로 위와 같은 에러가 발생하게 된다.

성능 데이터 저장을 위해 비즈니스 메서드의 트랜잭션 속성을 바꿔서는 안 된다. 어떻게 해결하면 좋을까?

## 비동기 적용

비동기 호출을 활용하면 된다.

트랜잭션 전파 속성이 어떻게 지정되어 있더라도 **비동기로 별도의 스레드에서 DB에 쓰기 작업을 수행하면 이 쓰기 작업은 별도의 트랜잭션에서 실행**되므로 위와 같은 에러를 우회할 수 있다. 그리고 당연히 항상 별도의 스레드를 생성하지 말고 **전용 스레드풀을 구성**하고 풀에서 스레드를 꺼내 쓰는 것이 좋다.

암튼 비동기 호출을 적용한 코드는 다음과 같다. 쿼리 파라미터를 매핑하는 private 메서드는 앞에 나온 코드와 동일하므로 생략한다.

또 한 가지 바뀐 점은 롬복의 `@RequiredArgsConstructor(onConstructor = @__(@Autowired))`를 이번에는 사용하지 않고 명시적으로 생성자를 작성했는데, 동일한 타입의 Bean이 있어서 `@Qualifer`로 구분하는 경우에는 이렇게 명시적으로 생성자를 작성해주지 않으면 Bean이 구별되지 않고 중복된다는 에러가 발생한다. 관련 내용은 [여기](https://github.com/HomoEfficio/dev-tips/blob/master/Lombok%20%40RequiredArgsConstructor(onConstructor%20%3D%20%40__(%40Autowired))%20주의%20사항.md)를 참고한다.

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

    @Around("execution(* a.b.c.d.e.f.g.ImpalaQueryExecutor.query*(..))")
    public Object onAroundImpalaQuery(ProceedingJoinPoint joinPoint) {

        long startTime = System.currentTimeMillis();
        Object[] args = joinPoint.getArgs();
        String queryTemplate = (String) args[0];
        Object[] queryParams = args.length == 2 ? (Object[]) args[1] : new Object[] {};
        String mappedQuery = getMappedQuery(queryTemplate, queryParams);
        Object result = null;

        try {
            Object queryResult = joinPoint.proceed();
            final long endTime = System.currentTimeMillis();
            result = queryResult;            
            this.taskExecutor.execute(
                    () -> this.repository.save(
                            new AsyncQueryResult(
                                    mappedQuery,
                                    queryResult instanceof List ? ((List) queryResult).size() : 1,
                                    new Date(startTime),
                                    new Date(endTime),
                                    (endTime - startTime) / 1000.0f,
                                    "OK"
                            )
                    )
            );

        } catch (Throwable throwable) {
            final long endTime = System.currentTimeMillis();
            this.taskExecutor.execute(
                    () -> this.repository.save(
                            new AsyncQueryResult(
                                    mappedQuery,
                                    -1,
                                    new Date(startTime),
                                    new Date(endTime),
                                    (endTime - startTime) / 1000.0f,
                                    "FAIL"
                            )
                    )
            );
            // 원래 메서드에서 처리하게 함
            throw new RuntimeException(throwable);
        }
        return result;
    }

    ...
}
```

