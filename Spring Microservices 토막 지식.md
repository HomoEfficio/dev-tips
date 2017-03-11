# Spring Microservices 토막 지식.md

- 대부분의 SOA 구현체가 서비스 수준의 추상화를 제공하는데 반해, 마이크로서비스는 한 발 더 나아가 실행 환경까지도 추상화한다.
- 마이크로서비스는 SOA에서 사용되던 ESB(Enterprise Service Bus) 같은 무거운 엔터프라이즈급 제품에 의존하지 않는다.
- SOA는 정적 레지스트리와 설정에 의존, 마이크로서비스는 동적 레지스트리 정보에 의존

- 관심의 초점은 평균 무고장 시간(MTBF, Mean Time Between Failure)에서 평균 복구 시간(Mean Time To Recovery)으로 이동하고 있다.

- 마이크로서비스의 중앙 집중식 로깅 구현
    - Splunk, Greylog, Logstash, Logplex, Loggly 같은 도구를 logback appender로 추가해서 로그가 중앙에 적재되도록 한다.


