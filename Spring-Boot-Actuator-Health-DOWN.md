# Spring Boot Actuator Health DOWN

Actuator를 포함하고 있는 스프링 부트 애플리케이션이 정상 기동됐음에도 불구하고 `/actuator/health`에 접속해보면 status가 DOWN으로 나올 때가 있다.

원인은 여러가지가 있는데 결국은 health 항목에 나오는 것 중 하나라도 DOWN이 있기 때문이다.  
그래서 다음과 같이 설정해서 `/actuator/health` 접속 시 상세 항목을 볼 수 있게 하고,

```yml
management:
  endpoint:
    health:
      show-details: always
```

화면에 표시되는 상세 내용 중에 DOWN인 놈을 찾아서 그 항목이 UP이 되게 조치하멷 된다.
