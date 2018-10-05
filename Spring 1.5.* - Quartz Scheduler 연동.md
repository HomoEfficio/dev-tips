# Spring 1.5.* Quartz 연동

다음과 같은 기준으로 연동하는 방법을 정리한다.

- NON_CLUSTERED 기준
- MySQL 기준
- JobStoreTX 기준

## Quartz 설정 파일 작성 및 로딩

### 설정 파일 작성

[설정 파일 매뉴얼](http://www.quartz-scheduler.org/documentation/quartz-2.2.x/configuration/)을 참고해서 다음과 같이 작성한다.

- NON_CLUSTERED 기준
- MySQL 기준
- JobStoreTX 기준

아래 내용은 기본값이 None으로 설정된 항목을 제외하고 필수, 옵션 관계 없이 모든 항목을 작성했으며, 매뉴얼 내용을 보고 필요하지 않은 옵션 값은 명시하지 않아도 된다.

특이하게도 아래와 같이 **숫자로 된 값들은 모두 따옴표로 문자열화 해줘야 오류가 발생하지 않는다.**

```
org.quartz:
  # http://www.quartz-scheduler.org/documentation/quartz-2.2.x/configuration/ConfigMain.html
  scheduler:
    instanceName: WhatEverName
    instanceId: NON_CLUSTERED
    instanceIdGenerator.class: org.quartz.simpl.SimpleInstanceIdGenerator
    threadName: WhatEverName_QuartzSchedulerThread
    makeSchedulerThreadDaemon: false
    threadsInheritContextClassLoaderOfInitializer: true
    idleWaitTime: '30000'
    dbFailureRetryInterval: '15000'
    classLoadHelper.class: org.quartz.simpl.CascadingClassLoadHelper
    jobFactory.class: org.quartz.simpl.PropertySettingJobFactory
    wrapJobExecutionInUserTransaction: false
    skipUpdateCheck: false
    batchTriggerAcquisitionMaxCount: '1'
    batchTriggerAcquisitionFireAheadTimeWindow: '0'
  # http://www.quartz-scheduler.org/documentation/quartz-2.2.x/configuration/ConfigThreadPool.html
  threadPool:
    class: org.quartz.simpl.SimpleThreadPool
    threadCount: '10'
    threadPriority: '5'
    makeThreadsDaemons: false
    threadsInheritGroupOfInitializingThread: false
    threadsInheritContextClassLoaderOfInitializingThread: true
  # http://www.quartz-scheduler.org/documentation/quartz-2.2.x/configuration/ConfigPlugins.html
  plugin:
    shutdownhook.class: org.quartz.plugins.management.ShutdownHookPlugin
    shutdownhook.cleanShutdown: true
  # http://www.quartz-scheduler.org/documentation/quartz-2.2.x/configuration/ConfigJobStoreTX.html
  jobStore:
    class: org.quartz.impl.jdbcjobstore.JobStoreTX
    driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
    dataSource: myDS
    tablePrefix: QRTZ_
    useProperties: false
    misfireThreshold: '60000'
    isClustered: false
    clusterCheckinInterval: '15000'
    maxMisfiresToHandleAtATime: '20'
    dontSetAutoCommitFalse: false
    selectWithLockSQL: SELECT * FROM {0}LOCKS WHERE SCHED_NAME = {1} AND LOCK_NAME = ? FOR UPDATE
    txIsolationLevelSerializable: false
    acquireTriggersWithinLock: false
  # http://www.quartz-scheduler.org/documentation/quartz-2.2.x/configuration/ConfigDataSources.html
  dataSource:
    myDS:
      driver: com.mysql.jdbc.Driver
      URL: jdbc:mysql://127.0.0.1:3306/WHATEVER_DB_NAME
      user: WHATEVER_USER_NAME
      password: WHATEVER_USER_NAME
      maxConnections: '5'
      validationQuery: SELECT 1
      idleConnectionValidationSeconds: '50'
      validateOnCheckout: false
#      discardIdleConnectionsSeconds: '0'
  context:
    key:
      COMMON_KEY: COMMON_VALUE  # 모든 Job에 공통으로 들어가야 할 항목이 있다면 이와 같이 작성
```

설정 내용은 `application.yml(properties)` 파일 내부에 두어도 되며, 별도의 파일로 저장해도 된다. 여기에서는 `resources/quartz/quartz.yml` 파일에 따로 저장한다.

### 설정 파일 로딩

다음과 같이 `@Configuration` 클래스 파일을 작성하면, 실제 Job을 스케줄링하는 스케줄러를 Bean으로 등록하면서 설정 내용이 로딩된다.

```java
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.impl.StdSchedulerFactory;
import org.springframework.beans.factory.config.YamlPropertiesFactoryBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;

import java.util.Properties;

@Configuration
public class QuartzConfig {

    @Bean
    public Scheduler statsModelScheduler() throws SchedulerException {
        StdSchedulerFactory stdSchedulerFactory = new StdSchedulerFactory();
        stdSchedulerFactory.initialize(quartzProperties());
        Scheduler scheduler = stdSchedulerFactory.getScheduler();
        scheduler.start();
        return scheduler;
    }

    private Properties quartzProperties() {
        YamlPropertiesFactoryBean yamlPropertiesFactoryBean = new YamlPropertiesFactoryBean();
        yamlPropertiesFactoryBean.setResources(new ClassPathResource("/quartz/quartz.yml"));
        yamlPropertiesFactoryBean.afterPropertiesSet();
        return yamlPropertiesFactoryBean.getObject();
    }
}
```

## 테이블 생성

http://www.quartz-scheduler.org/downloads/ 에서 받은 파일 quartz 파일의 압축을 풀면 `/docs/dbTables` 폴더에 지원되는 DB별 테이블 생성 sql 파일을 확인할 수 있다.

여기에서는 MySQL을 사용하므로 MySQL 클라이언트 프로그램을 통해 sql 파일 내용대로 테이블을 생성한다.

## 기타

- `scheduler.scheduleJob()`이 호출되어야 `QUARTZ_*` 테이블에 데이터가 저장된다.
- `QUARTZ_*` 테이블에 데이터가 저장된 Job은 Quartz 애플리케이션 재시작 시 테이블에서 Job 관련 정보를 읽어서 자동으로 재스케줄링 하므로, 배포 후에 별도로 스케줄링 해주지 않아도 저장된 내용에 따라 스케줄링 된다.
  - 정확하진 않지만 `scheduler.start()`가 호출되면 `JobStoreSupport.schedulerStarted()`에서 DB에 저장된 정보를 읽어서 자동으로 스케줄링 하는 것으로 추정


