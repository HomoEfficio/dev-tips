# QueryDSL로 JPA 튜닝

JPA는 사용하기 정말 편하지만, 여기 저기 불필요한 객체 참조가 난무한 상태에서 편하게만 사용하다보면 성능에 악영향을 주는 쿼리가 넘쳐나게 된다.

그렇다고 이미 여기 사용되고 있는 객체 참조를 끊자니 일이 꽤 커진다. 언젠가는 끊어야겠지만 그 언젠가가 지금일 수는 없는 상황에서는 일단 해당 부분만 튜닝을 하는 수 밖에 없다.

이럴 때 QueryDSL이 출동하면 어떨까.

## 튜닝 전

튜닝 전의 상태는 대략 다음과 같다.

```sql
2019-11-21 15:39:44.250  org.hibernate.SQL                        : 
    select
        targettask1_.full_deploy_tasks_id as col_0_0_,
        target0_.name as col_1_0_,
        target0_.description as col_2_0_ 
    from
        target target0_,
        target_tasks targettask1_ 
    where
        target0_.target_tasks_id=targettask1_.id 
        and target0_.full_deploy_schedule_id=?
2019-11-21 15:39:44.262  org.hibernate.SQL                        : 
    select
        schedulehi0_.id as col_0_0_,
        schedulehi0_.id as col_1_0_,
        schedulehi0_.schedule_id as col_2_0_,
        schedulehi0_.tasks_id as col_3_0_,
        schedulehi0_.target_stats_id as col_4_0_,
        schedulehi0_.start_time as col_5_0_,
        schedulehi0_.end_time as col_6_0_,
        schedulehi0_.status as col_7_0_,
        schedule1_.id as id1_73_1_,
        tasks2_.id as id1_116_2_,
        targetstat3_.id as id1_108_3_,
        schedulehi0_.id as id1_75_0_,
        schedule1_.id as id1_73_4_,
        tasks2_.id as id1_116_5_,
        targetstat3_.id as id1_108_6_,
        schedulehi0_.created_date as created_2_75_0_,
        schedulehi0_.last_modified_date as last_mod3_75_0_,
        schedulehi0_.elapsed_minutes as elapsed_4_75_0_,
        schedulehi0_.end_time as end_time5_75_0_,
        schedulehi0_.message as message6_75_0_,
        schedulehi0_.schedule_id as schedule9_75_0_,
        schedulehi0_.start_time as start_ti7_75_0_,
        schedulehi0_.status as status8_75_0_,
        schedulehi0_.target_stats_id as target_10_75_0_,
        schedulehi0_.tasks_id as tasks_i11_75_0_,
        schedulehi0_.trait_target_stats_id as trait_t12_75_0_,
        schedule1_.created_date as created_2_73_1_,
        schedule1_.last_modified_date as last_mod3_73_1_,
        schedule1_.cron_expression as cron_exp4_73_1_,
        schedule1_.interval_sec as interval5_73_1_,
        schedule1_.schedule_hour as schedule6_73_1_,
        schedule1_.schedule_minutue as schedule7_73_1_,
        schedule1_.schedule_type as schedule8_73_1_,
        schedule1_.status as status9_73_1_,
        tasks2_.created_date as created_2_116_2_,
        tasks2_.last_modified_date as last_mod3_116_2_,
        tasks2_.description as descript4_116_2_,
        tasks2_.is_reporting as is_repor5_116_2_,
        tasks2_.name as name6_116_2_,
        tasks2_.reporting_email_address as reportin7_116_2_,
        tasks2_.status as status8_116_2_,
        targetstat3_.created_date as created_2_108_3_,
        targetstat3_.last_modified_date as last_mod3_108_3_,
        targetstat3_.cookie_audience_count as cookie_a4_108_3_,
        targetstat3_.gaid_audience_count as gaid_aud5_108_3_,
        targetstat3_.idfa_audience_count as idfa_aud6_108_3_,
        targetstat3_.read_end_time as read_end7_108_3_,
        targetstat3_.read_start_time as read_sta8_108_3_,
        targetstat3_.schedule_id as schedul15_108_3_,
        targetstat3_.send_end_time as send_end9_108_3_,
        targetstat3_.send_start_time as send_st10_108_3_,
        targetstat3_.start_time as start_t11_108_3_,
        targetstat3_.status as status12_108_3_,
        targetstat3_.target_id as target_16_108_3_,
        targetstat3_.total_data_size as total_d13_108_3_,
        targetstat3_.unique_audience_count as unique_14_108_3_,
        schedule1_.created_date as created_2_73_4_,
        schedule1_.last_modified_date as last_mod3_73_4_,
        schedule1_.cron_expression as cron_exp4_73_4_,
        schedule1_.interval_sec as interval5_73_4_,
        schedule1_.schedule_hour as schedule6_73_4_,
        schedule1_.schedule_minutue as schedule7_73_4_,
        schedule1_.schedule_type as schedule8_73_4_,
        schedule1_.status as status9_73_4_,
        tasks2_.created_date as created_2_116_5_,
        tasks2_.last_modified_date as last_mod3_116_5_,
        tasks2_.description as descript4_116_5_,
        tasks2_.is_reporting as is_repor5_116_5_,
        tasks2_.name as name6_116_5_,
        tasks2_.reporting_email_address as reportin7_116_5_,
        tasks2_.status as status8_116_5_,
        targetstat3_.created_date as created_2_108_6_,
        targetstat3_.last_modified_date as last_mod3_108_6_,
        targetstat3_.cookie_audience_count as cookie_a4_108_6_,
        targetstat3_.gaid_audience_count as gaid_aud5_108_6_,
        targetstat3_.idfa_audience_count as idfa_aud6_108_6_,
        targetstat3_.read_end_time as read_end7_108_6_,
        targetstat3_.read_start_time as read_sta8_108_6_,
        targetstat3_.schedule_id as schedul15_108_6_,
        targetstat3_.send_end_time as send_end9_108_6_,
        targetstat3_.send_start_time as send_st10_108_6_,
        targetstat3_.start_time as start_t11_108_6_,
        targetstat3_.status as status12_108_6_,
        targetstat3_.target_id as target_16_108_6_,
        targetstat3_.total_data_size as total_d13_108_6_,
        targetstat3_.unique_audience_count as unique_14_108_6_ 
    from
        schedule_history schedulehi0_ 
    inner join
        schedule schedule1_ 
            on schedulehi0_.schedule_id=schedule1_.id 
    inner join
        tasks tasks2_ 
            on schedulehi0_.tasks_id=tasks2_.id 
    left outer join
        target_statistics targetstat3_ 
            on schedulehi0_.target_stats_id=targetstat3_.id 
    where
        schedulehi0_.schedule_id=? 
        and (
            schedulehi0_.end_time>? 
            and schedulehi0_.end_time<? 
            or schedulehi0_.start_time>? 
            and schedulehi0_.start_time<?
        )
2019-11-21 15:39:44.288  org.hibernate.SQL                        : 
    select
        target0_.id as id1_102_0_,
        target0_.created_date as created_2_102_0_,
        target0_.last_modified_date as last_mod3_102_0_,
        target0_.description as descript4_102_0_,
        target0_.diff_deploy_schedule_id as diff_dep9_102_0_,
        target0_.full_deploy_schedule_id as full_de10_102_0_,
        target0_.name as name5_102_0_,
        target0_.reporting_email_address as reportin6_102_0_,
        target0_.site_id as site_id11_102_0_,
        target0_.status as status7_102_0_,
        target0_.target_tasks_id as target_12_102_0_,
        target0_.type as type8_102_0_,
        schedule1_.id as id1_73_1_,
        schedule1_.created_date as created_2_73_1_,
        schedule1_.last_modified_date as last_mod3_73_1_,
        schedule1_.cron_expression as cron_exp4_73_1_,
        schedule1_.interval_sec as interval5_73_1_,
        schedule1_.schedule_hour as schedule6_73_1_,
        schedule1_.schedule_minutue as schedule7_73_1_,
        schedule1_.schedule_type as schedule8_73_1_,
        schedule1_.status as status9_73_1_,
        scheduleda2_.schedule_id as schedule1_74_2_,
        scheduleda2_.schedule_day_of_week_list as schedule2_74_2_,
        schedule3_.id as id1_73_3_,
        schedule3_.created_date as created_2_73_3_,
        schedule3_.last_modified_date as last_mod3_73_3_,
        schedule3_.cron_expression as cron_exp4_73_3_,
        schedule3_.interval_sec as interval5_73_3_,
        schedule3_.schedule_hour as schedule6_73_3_,
        schedule3_.schedule_minutue as schedule7_73_3_,
        schedule3_.schedule_type as schedule8_73_3_,
        schedule3_.status as status9_73_3_,
        site4_.id as id1_90_4_,
        site4_.created_date as created_2_90_4_,
        site4_.last_modified_date as last_mod3_90_4_,
        site4_.description as descript4_90_4_,
        site4_.interwork_type as interwor5_90_4_,
        site4_.is_admin as is_admin6_90_4_,
        site4_.name as name7_90_4_,
        site4_.site_code as site_cod8_90_4_,
        site4_.status as status9_90_4_,
        site4_.url as url10_90_4_,
        site4_.zone_id as zone_id11_90_4_,
        targettask5_.id as id1_112_5_,
        targettask5_.created_date as created_2_112_5_,
        targettask5_.last_modified_date as last_mod3_112_5_,
        targettask5_.description as descript4_112_5_,
        targettask5_.diff_deploy_tasks_id as diff_dep7_112_5_,
        targettask5_.full_deploy_tasks_id as full_dep8_112_5_,
        targettask5_.name as name5_112_5_,
        targettask5_.status as status6_112_5_,
        tasks6_.id as id1_116_6_,
        tasks6_.created_date as created_2_116_6_,
        tasks6_.last_modified_date as last_mod3_116_6_,
        tasks6_.description as descript4_116_6_,
        tasks6_.is_reporting as is_repor5_116_6_,
        tasks6_.name as name6_116_6_,
        tasks6_.reporting_email_address as reportin7_116_6_,
        tasks6_.status as status8_116_6_,
        tasks7_.id as id1_116_7_,
        tasks7_.created_date as created_2_116_7_,
        tasks7_.last_modified_date as last_mod3_116_7_,
        tasks7_.description as descript4_116_7_,
        tasks7_.is_reporting as is_repor5_116_7_,
        tasks7_.name as name6_116_7_,
        tasks7_.reporting_email_address as reportin7_116_7_,
        tasks7_.status as status8_116_7_ 
    from
        target target0_ 
    left outer join
        schedule schedule1_ 
            on target0_.diff_deploy_schedule_id=schedule1_.id 
    left outer join
        schedule_schedule_day_of_week_list scheduleda2_ 
            on schedule1_.id=scheduleda2_.schedule_id 
    left outer join
        schedule schedule3_ 
            on target0_.full_deploy_schedule_id=schedule3_.id 
    left outer join
        site site4_ 
            on target0_.site_id=site4_.id 
    left outer join
        target_tasks targettask5_ 
            on target0_.target_tasks_id=targettask5_.id 
    left outer join
        tasks tasks6_ 
            on targettask5_.diff_deploy_tasks_id=tasks6_.id 
    left outer join
        tasks tasks7_ 
            on targettask5_.full_deploy_tasks_id=tasks7_.id 
    where
        target0_.id=?
2019-11-21 15:39:44.317  org.hibernate.SQL                        : 
    select
        schedulehi0_.id as id1_75_0_,
        schedulehi0_.created_date as created_2_75_0_,
        schedulehi0_.last_modified_date as last_mod3_75_0_,
        schedulehi0_.elapsed_minutes as elapsed_4_75_0_,
        schedulehi0_.end_time as end_time5_75_0_,
        schedulehi0_.message as message6_75_0_,
        schedulehi0_.schedule_id as schedule9_75_0_,
        schedulehi0_.start_time as start_ti7_75_0_,
        schedulehi0_.status as status8_75_0_,
        schedulehi0_.target_stats_id as target_10_75_0_,
        schedulehi0_.tasks_id as tasks_i11_75_0_,
        schedulehi0_.trait_target_stats_id as trait_t12_75_0_ 
    from
        schedule_history schedulehi0_ 
    where
        schedulehi0_.target_stats_id=?
2019-11-21 15:39:44.322  org.hibernate.SQL                        : 
    select
        sourceidty0_.target_id as target_i1_105_0_,
        sourceidty0_.label as label2_105_0_,
        sourceidty0_.value as value3_105_0_ 
    from
        target_source_id_type_list sourceidty0_ 
    where
        sourceidty0_.target_id=?
2019-11-21 15:39:44.327  org.hibernate.SQL                        : 
    select
        idsourcety0_.target_id as target_i1_104_0_,
        idsourcety0_.id_source_type_list as id_sourc2_104_0_ 
    from
        target_id_source_type_list idsourcety0_ 
    where
        idsourcety0_.target_id=?
2019-11-21 15:39:44.329  org.hibernate.SQL                        : 
    select
        categoryid0_.target_id as target_i1_103_0_,
        categoryid0_.category_id_list as category2_103_0_ 
    from
        target_category_id_list categoryid0_ 
    where
        categoryid0_.target_id=?
2019-11-21 15:39:44.334  org.hibernate.SQL                        : 
    select
        successseg0_.target_statistics_id as target_s1_110_0_,
        successseg0_.success_segment_audience_count_map as success_2_110_0_,
        successseg0_.success_segment_audience_count_map_key as success_3_0_ 
    from
        target_statistics_success_segment_audience_count_map successseg0_ 
    where
        successseg0_.target_statistics_id=?
2019-11-21 15:39:44.342  org.hibernate.SQL                        : 
    select
        failedsegm0_.target_statistics_id as target_s1_109_0_,
        failedsegm0_.failed_segment_audience_count_map as failed_s2_109_0_,
        failedsegm0_.failed_segment_audience_count_map_key as failed_s3_0_ 
    from
        target_statistics_failed_segment_audience_count_map failedsegm0_ 
    where
        failedsegm0_.target_statistics_id=?
2019-11-21 15:39:44.392  org.hibernate.SQL                        : 
    select
        jobaverage0_.schedule_id as schedule1_36_,
        jobaverage0_.created_date as created_2_36_,
        jobaverage0_.last_modified_date as last_mod3_36_,
        jobaverage0_.average_minutes as average_4_36_ 
    from
        job_average_execution_time jobaverage0_ 
    where
        jobaverage0_.schedule_id=?
2019-11-21 15:39:44.408  org.hibernate.SQL                        : 
    select
        jobaverage0_.schedule_id as schedule1_36_,
        jobaverage0_.created_date as created_2_36_,
        jobaverage0_.last_modified_date as last_mod3_36_,
        jobaverage0_.average_minutes as average_4_36_ 
    from
        job_average_execution_time jobaverage0_ 
    where
        jobaverage0_.schedule_id=?
```

이보다 심한 사례도 얼마든지 많겠지만 여튼 현 상태는 이렇고, 굳이 없어도 되는 여러 엔티티의 필드값까지 모두 가져오게 돼 있어서 join도 많고 불필요한 쿼리 자체도 많다. 그리고 이런 쿼리가 한 번만 수행되는 게 아니라 백 번 내외로 수행된다.

이를 바로 잡기 위한 방법은 여러 가지가 있겠지만 가장 직관적이고 간편한 방법 중의 하나는 QueryDSL로 Projection(필요한 필드만 추출)을 적용하는 방법이다.

## QueryDSL Projection 적용

QueryDSL 사용을 위한 의존 관계나 플러그인 설정 등은 돼 있다고 보면 다음에 해 줄 설정은 간단하다.

### JPAQueryFactory 설정

```java
@Configuration
public class QueryDslConfig {

    @PersistenceContext
    private EntityManager entityManager;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

위와 같이 `JPAQueryFactory`를 빈으로 등록해주면 Repository Custom 구현체에서 `JPAQueryFactory`를 주입 받아서 QueryDSL 을 사용할 수 있다.

### Projection DTO 작성

필요한 필드만 쏙 빼낸 Projection DTO를 작성한다.

```java
@Getter
@AllArgsConstructor
public class ScheduleHistoryDto {

    private Long scheduleHistoryId;
    private Long scheduleId;        // 원래는 객체 참조라서 join 을 유발하던 필드
    private Long tasksId;           // 원래는 객체 참조라서 join 을 유발하던 필드

    private Long totalDataSize;
    private Integer uniqueAudienceCount;
    private Integer gaidAudienceCount;
    private Integer idfaAudienceCount;
    private Integer cookieAudienceCount;

    private LocalDateTime startTime;
    private LocalDateTime endTime;

    private ExecutionStatus status;
}
```

### Custom Repository 인터페이스 작성

먼저 QueryDSL 을 사용해서 구현될 Custom Repository 인터페이스를 작성한다. 앞에서 작성한 Projection DTO 타입으로 결과를 반환한다.

나중에 다른 엔티티의 Repository 가 이 Custom Repository 를 상속받게 해서 QueryDSL로 최적화 된 쿼리를 사용할 수 있게 된다.

```java
public interface ScheduleHistoryRepositoryCustom {

    List<ScheduleHistoryDto> findScheduleHistoryByScheduleId(Long scheduleId, LocalDateTime dateStartTime, LocalDateTime dateEndTime);
}
```

### Custom Repository 인터페이스 구현체 작성

QueryDSL과 Spring Data JPA 규약과 관련이 있는데, 이 구현체는 이름이 중요하다. 본 예제를 기준으로 원래 `ScheduleHistoryRepository` 를 사용 중이었다면, Custom Repository 구현체의 이름은 `ScheduleHistoryRepositoryImpl` 이어야 한다. 실제 구현 대상은 `ScheduleHistoryRepositoryCustom` 이지만 그렇다고 `ScheduleHistoryRepositoryCustomImpl` 이라고 이름 지으면 나중에 `ScheduleHistoryRepository`에서는 추가된 기능을 사용할 수 없다.

작성 방법은 QueryDSL 사용법 외에는 그리 특별한 것은 없다. 특히 `Projections.constructor(Dto.class, field1, field2, ...)`를 통해 Projection 이 이루어진다는 점에 주목

아래 코드에서는 `leftJoin()`이 있는데, `targetStats` 값이 없을 때도 `scheduleHistory` 결과가 있어야 하므로 불가피하게 추가되었을 뿐이며, `schedule`이나 `tasks` 같은 다른 연관 관계는 별도로 `join()` 을 지정하지 않아도 결국 inner join 으로 조회된다.

```java
@RequiredArgsConstructor
public class ScheduleHistoryRepositoryImpl implements ScheduleHistoryRepositoryCustom {

    private final JPAQueryFactory jpaQueryFactory;

    @Override
    public List<ScheduleHistoryDto> findScheduleHistoryByScheduleId(Long scheduleId, LocalDateTime dateStartTime, LocalDateTime dateEndTime) {
        QScheduleHistory scheduleHistory = QScheduleHistory.scheduleHistory;
        return jpaQueryFactory.select(
                Projections.constructor(ScheduleHistoryDto.class,
                        scheduleHistory.id,
                        scheduleHistory.schedule.id,
                        scheduleHistory.tasks.id,
                        scheduleHistory.targetStats.totalDataSize,
                        scheduleHistory.targetStats.uniqueAudienceCount,
                        scheduleHistory.targetStats.gaidAudienceCount,
                        scheduleHistory.targetStats.idfaAudienceCount,
                        scheduleHistory.targetStats.cookieAudienceCount,
                        scheduleHistory.startTime,
                        scheduleHistory.endTime,
                        scheduleHistory.status))
                .from(scheduleHistory)
                .leftJoin(scheduleHistory.targetStats)
                .where(scheduleHistory.schedule.id.eq(scheduleId),
                        (scheduleHistory.endTime.after(dateStartTime).and(scheduleHistory.endTime.before(dateEndTime)))
                                .or(scheduleHistory.startTime.after(dateStartTime).and(scheduleHistory.startTime.before(dateEndTime))))
                .fetch();
    }
}

```

### 기존 Repository 인터페이스가 Custom Repository 를 상속받도록 변경

아래와 같이 QueryDSL로 구현된 구현체가 실체화(realization)하는 Custom Repository 인 `ScheduleHistoryRepositoryCustom`를 상속받게 한다.

```java
public interface ScheduleHistoryRepository extends JpaRepository<ScheduleHistory, Long>, ScheduleHistoryRepositoryCustom {

   ...

}
```

### QueryDSL로 구현된 메서드 호출

다음과 같이 `ScheduleHistoryRepositoryCustom`가 아닌 `ScheduleHistoryRepository`를 통해 호출한다.

```java
List<ScheduleHistoryDto> alreadyStartedScheduleHistoryViews = scheduleHistoryRepository.findScheduleHistoryByScheduleId(
                scheduleId,
                jobQueryMeta.getQueryDateStartTime(),
                jobQueryMeta.getQueryDateEndTime());
```

## 튜닝 후

결과는 다음과 같이 깔끔해진다.

```sql
2019-11-22 12:17:03.349 org.hibernate.SQL                        : 
    select
        targettask1_.full_deploy_tasks_id as col_0_0_,
        target0_.name as col_1_0_,
        target0_.description as col_2_0_ 
    from
        target target0_,
        target_tasks targettask1_ 
    where
        target0_.target_tasks_id=targettask1_.id 
        and target0_.full_deploy_schedule_id=?
2019-11-22 12:17:03.355 org.hibernate.SQL                        : 
    select
        schedulehi0_.id as col_0_0_,
        schedulehi0_.schedule_id as col_1_0_,
        schedulehi0_.tasks_id as col_2_0_,
        targetstat1_.total_data_size as col_3_0_,
        targetstat1_.unique_audience_count as col_4_0_,
        targetstat1_.gaid_audience_count as col_5_0_,
        targetstat1_.idfa_audience_count as col_6_0_,
        targetstat1_.cookie_audience_count as col_7_0_,
        schedulehi0_.start_time as col_8_0_,
        schedulehi0_.end_time as col_9_0_,
        schedulehi0_.status as col_10_0_ 
    from
        schedule_history schedulehi0_ 
    left outer join
        target_statistics targetstat1_ 
            on schedulehi0_.target_stats_id=targetstat1_.id 
    where
        schedulehi0_.schedule_id=? 
        and (
            schedulehi0_.end_time>? 
            and schedulehi0_.end_time<? 
            or schedulehi0_.start_time>? 
            and schedulehi0_.start_time<?
        )
2019-11-22 12:17:03.363 org.hibernate.SQL                        : 
    select
        jobaverage0_.schedule_id as schedule1_36_,
        jobaverage0_.created_date as created_2_36_,
        jobaverage0_.last_modified_date as last_mod3_36_,
        jobaverage0_.average_minutes as average_4_36_ 
    from
        job_average_execution_time jobaverage0_ 
    where
        jobaverage0_.schedule_id=?
2019-11-22 12:17:03.617 org.hibernate.SQL                        : 
    select
        jobaverage0_.schedule_id as schedule1_36_,
        jobaverage0_.created_date as created_2_36_,
        jobaverage0_.last_modified_date as last_mod3_36_,
        jobaverage0_.average_minutes as average_4_36_ 
    from
        job_average_execution_time jobaverage0_ 
    where
        jobaverage0_.schedule_id=?
```

## 한 걸음 더

위와 같이 쿼리를 튜닝해서도 많이 개선되었지만 정말로 수행 시간을 많이 잡아먹는 부분은 의외로 다음과 같이 불필요하게 반복 조회하는 쿼리였다. 여기에는 `?`로 표시돼있는 파라미터 값도 동일하게 반복된다.

```sql
2019-11-22 12:17:03.363 org.hibernate.SQL                        : 
    select
        jobaverage0_.schedule_id as schedule1_36_,
        jobaverage0_.created_date as created_2_36_,
        jobaverage0_.last_modified_date as last_mod3_36_,
        jobaverage0_.average_minutes as average_4_36_ 
    from
        job_average_execution_time jobaverage0_ 
    where
        jobaverage0_.schedule_id=?
2019-11-22 12:17:03.617 org.hibernate.SQL                        : 
    select
        jobaverage0_.schedule_id as schedule1_36_,
        jobaverage0_.created_date as created_2_36_,
        jobaverage0_.last_modified_date as last_mod3_36_,
        jobaverage0_.average_minutes as average_4_36_ 
    from
        job_average_execution_time jobaverage0_ 
    where
        jobaverage0_.schedule_id=?
```

경우에 따라서는 위 쿼리가 약 300회 호출되는 경우도 있었다. 이는 다음과 같이 그냥 `LinkedHashMap`으로 캐시를 만들어 사용하니 성능이 대폭 개선되었다.

### 캐시 적용 전

```java
    private Double getAvgElapsedMinutesOfSchedule(Schedule schedule) {
        return jobAverageExecutionTimeRepository.findByScheduleId(schedule.getId())
                    .map(JobAverageExecutionTime::getAverageMinutes)
                    .orElse(1.0d);
    }
```

### 캐시 적용 후

```java
    private Double getAvgElapsedMinutesOfSchedule(Schedule schedule) {
        Long scheduleId = schedule.getId();
        return avgElapsedTimeMap.computeIfAbsent(scheduleId,
                sid -> jobAverageExecutionTimeRepository.findByScheduleId(sid)
                        .map(JobAverageExecutionTime::getAverageMinutes)
                        .orElse(1.0d));
    }
```

