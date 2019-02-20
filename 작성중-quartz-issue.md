## Clearly describe your problems.

Jobs which were not marked for recovery are still being recovered by other quartz instances in the clustering mode with 3 nodes in quartz 2.3.0.

- Running Quartz clustering with 3 nodes of Quartz Version 2.3.0
- No2 Instance went bad and No1 Instance recovered(reran) the jobs which was running in No2
![dmp-quartz-recovery-log-toquartz](https://user-images.githubusercontent.com/17228983/52398221-b72f5080-2afb-11e9-9735-05c272a561db.png)
- But the jobs were not marked for recovery
![dmp-quartz-clustering-error-toquartz](https://user-images.githubusercontent.com/17228983/52398630-1346a480-2afd-11e9-8145-89c09eb9487e.png)
- So the jobs of No2 must not be rerun by other instances, but they are rerun.

## Tell us the workload of your quartz scheduler (how many jobs do you have running, how long does each job run on avg etc.)

- About 30 jobs are scheduled and running.
- The longest job takes more than 4 hours.
- One daily job is dealing with over 8G file I/O and takes over 3 hours and the quartz instance that processes this job goes bad from time to time.
- The shortest job takes a minute.

## Provide Quartz version, properties file used, DB info if there is any, and LOG output at DEBUG level if possible.

- version: 2.3.0
- properties file
```
org.quartz:
  scheduler:
    instanceName: DMPClusteredScheduler
    instanceId: AUTO
  threadPool:
    class: org.quartz.simpl.SimpleThreadPool
    threadCount: '25'
    threadPriority: '5'
  jobStore:
    misfireThreshold: '60000'
    class: org.quartz.impl.jdbcjobstore.JobStoreTX
    driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
    useProperties: false
    dataSource: myDs
    tablePrefix: QRTZ_
    isClustered: true
  dataSource:
    myDs:
      driver: com.mysql.jdbc.Driver
      URL: jdbc:mysql://XXXX:3306/quartz
      user: quartz_user
      password: "XXXX"
      maxConnections: 5
      validationQuery: select 1      
```
- DB info: MySQL 5.7
- 

## Provide a standalone test case to reproduce the problem if you can.

## Use dummy Job implementation. We will not able to help you debug your own custom job problems!

## Do not submit test case with other libraries mixed in. Provide standalone Quartz test case only. We will not able to help you debug other libraries (eg: Spring) that uses Quartz.

