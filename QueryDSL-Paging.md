# QueryDSL Paging

- `QueryDslRepositorySupport`를 사용하면 `getQueryDsl().applyPagination()`을 사용해서 페이징을 간단하게 적용할 수 있고,
- `PageableExecutionUtils.getPage()`를 사용해서 카운트 쿼리가 필요 없는 경우(첫 페이지로 모두 커버 되는 경우나 offset+content 사이즈로 전체 사이즈를 구할 수 있는 마지막 페이지인 경우)에는 카운트 쿼리를 실행하지 않도록 최적화 할 수 있다.

```java
public class ScheduleHistoryRepositoryImpl extends QueryDslRepositorySupport implements ScheduleHistoryRepositoryCustom {
    ...
    
    public Page<ScheduleHistorySimpleDto> findAllByTasksId(Long tasksId, Pageable pageable) {
        JPQLQuery<ScheduleHistorySimpleDto> query = from(scheduleHistory)
                .where(scheduleHistory.tasks.id.eq(tasksId))
                .orderBy(scheduleHistory.id.desc())
                .select(
                        Projections.constructor(ScheduleHistorySimpleDto.class,
                                scheduleHistory.id,
                                scheduleHistory.schedule.id,
                                scheduleHistory.tasks.id,
                                scheduleHistory.startTime,
                                scheduleHistory.endTime,
                                scheduleHistory.status,
                                scheduleHistory.message)
                )
                .fetchAll();
        List<ScheduleHistorySimpleDto> list = getQuerydsl().applyPagination(pageable, query).fetch();
        return PageableExecutionUtils.getPage(list, pageable, query::fetchCount);
    }
```
