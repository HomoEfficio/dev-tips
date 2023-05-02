# Spring JPA deleteAllById() OutOfMemory Error

다음과 같이 평범한 Spring Data Repository의 delete 문을 실행했는데,

```java
taskHistoryRepository.deleteAllByScheduleId(scheduleId);
```

아래와 같이 OOM 이 발생했다.

![Imgur](https://i.imgur.com/H5LPq2K.png)

테이블을 보니 아래와 같이 1.8G에 약 393만행..

![Imgur](https://i.imgur.com/lm13jEM.png)

지울 id에 해당하는 행은 약 2천 행 정도인데, 이 정도면 OOM이 발생하나보다.

테이블의 크기가 OOM의 직접적인 원인이라기보다는 삭제할 행의 수가 많은 게 OOM의 원인일 것이다.

delete 문에 limit 을 사용할 수 있는 DB(ex: MySQL)라면 간단한 해결 방법은 한꺼번에 지우지 말고 `delete ... limit ...` 문을 활용해서 쪼개서 지우는 것이다. 이 방법은 Spring Data 에서 지원해주지 않으며 native query를 사용해야 한다.

```java
// Repository
@Modifying
@Query(nativeQuery = true, value = "delete from task_history where schedule_id = :scheduleId limit :count")
int deleteAllByScheduleFirstN(@Param("scheduleId") Long scheduleId, @Param("count") int count);

// Service
int deletedRows = 1;
while (deletedRows > 0) {
    deletedRows = taskHistoryRepository.deleteAllByScheduleFirstN(scheduleId, 50);  // 50개씩 나눠서 삭제
}
```
