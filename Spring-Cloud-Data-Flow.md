# Spring Cloud Data Flow

# 주요 줄기

![Imgur](https://i.imgur.com/xRRwhpg.png)

1. Spring Cloud Stream 이나 Spring Cloud Task 또는 Spring Batch로 Stream Job이나 Batch Job 작성
1. 1 에서 만든 Job을 Client(Web Dashboard 또는 Shell)를 통해 Data Flow Server에 등록, 실행 및 관리
1. Data Flow Server는 
    - 등록된 Job을 Local, CloudFoundry 또는 Kubernetes 환경에 deploy
      - Batch Job은 직접 deploy, Stream Job은 Skipper 서버에 위임
    - Batch Job의 경우 Scheduling 지원
    - 실행 중인 Job 실행 상태 및 이력 관리
      - BatchJob의 경우 Spring Cloud Task가 `@EnableTask` 애너테이션을 통해 실행 시간 기록, 종료 코드 에러 메시지, 다른 app에 notify 등 기능 지원

# Batch Job 오케스트레이션

![Imgur](https://i.imgur.com/3i8EYCa.png)

- Spring Cloud Data Flow 서버를 통해 Job을 일회성 또는 정기적으로 실행 가능

## 배포 설정

![Imgur](https://i.imgur.com/i94gmiU.png)

실행 옵션 정보 지정 가능

## 스케줄링

https://dataflow.spring.io/docs/feature-guides/batch/scheduling/ 참고

### 주요 줄기

![Imgur](https://i.imgur.com/F10zTJx.png)

- Client를 통해 Spring Cloud Data Flow 서버에서 스케줄링을 등록하면
- Scheduling Agent(k8s의 CronJob)가 정해진 시간에 `SchedulerTaskLauncher`를 실행
- `SchedulerTaskLauncher`는 SCDF 서버의 REST API를 호출해서 관련 데이터를 가져오고 실제 Job을 실행한다. 

### 주의 사항

> Spring Cloud Data Flow 는  
> CloudFoundry나 Kubernetes 환경에서만 스케줄링을 지원하며,  
> Local 환경에서는 스케줄링은 불가하며 즉시 실행만 가능하다.

- Spring Cloud Data Flow에서 스케줄링 기능까지 사용하려면 결국 CloudFoundry나 Kubernetes 환경이 필요
- Docker 기반의 Kubernetes 경우 **Java 8은 세부 버전에 따라 자원 할당 관련 이슈가 있으므로 확인 필요**
  - **Java 10 이상에서는 이 문제가 해결**돼있다.
  - 참고: https://blog.softwaremill.com/docker-support-in-new-java-8-finally-fd595df0ca54

### 스케줄링 등록

![Imgur](https://i.imgur.com/vnhr8jN.png)

![Imgur](https://i.imgur.com/NybeLvY.png)

- Task에 등록된 Job의 스케줄링 지정

### 스케줄링 해제

![Imgur](https://i.imgur.com/gKXos0V.png)

### 스케줄링 이력 조회

![Imgur](https://i.imgur.com/FSgmBVD.png)

# Task 모니터링

https://dataflow.spring.io/docs/feature-guides/batch/monitoring/ 참고

Prometheus나 InfluxDB와 연동해서 모니터링 지원


