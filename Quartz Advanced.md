# Quartz Advanced

# Quartz를 사용하는 이유

- Clustering
- 트리거를 통한 Job 상태 관리
    - 스케줄링 정보 유지
    - misfire 처리

# Spring Boot 통합

- Job 클래스를 Bean으로 등록하는 과정

# Job 클래스 외부화

- CustomClassLoader 설정
- CustomClassLoader 구현

# 예외 처리

- 강제 종료: 절대 Job을 구현하지 말고 반드시 InterruptableJob 을 구현해서 강제 종료 가능하게 해야함
  - Trigger.triggerComplete () 내부에서 예외 발생 시 해당 trigger가 계속 `BLOCKED` 상태로 남아있는 오류 있음
  - InterruptableJob이 구현된 Job만 강제로 Kill 해서 `BLOCKED` 상태 해제 가능
  - 기타 여러 이유로 `BLOCKED` 상태로 남아있는 trigger를 Kill 하기 위해 반드시 필요
- 비정상 예외

# 워크플로우

- 스케줄링 병렬화와 실행 스레드의 얽힘  

