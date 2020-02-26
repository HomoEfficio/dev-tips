# SQL Update Table Join 4 Tables Case When

하나의 테이블의 컬럼 하나를 업데이트 하는 데 4개의 테이블이 동원되고, CASE WHEN 을 사용하는 사례

where 없이도 행 별로 맞는 데이터로 업데이트 된다.

```sql
alter table `target` add `transfer_type` varchar(255) NULL after `name`
;

update `target` d
    inner join `target_tasks` c on c.id = d.target_tasks_id
    inner join `tasks` b on b.id = c.full_deploy_tasks_id
    inner join `task` a on a.tasks_id = b.id
set d.`transfer_type` =
        case
            when a.job_class_name like '%transfer%' and a.job_class_name like '%AWSS3%' then 'AWS_S3'
            when a.job_class_name is null or a.job_class_name = '' then null
            else 'CUSTOM'
            end
;
```
