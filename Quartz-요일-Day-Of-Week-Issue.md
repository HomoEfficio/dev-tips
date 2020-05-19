# Quartz 요일 Day Of Week 문제

Java의 요일은 https://docs.oracle.com/javase/8/docs/api/java/time/DayOfWeek.html 여기에 있는데,  
ISO-8601 에 따라 월 ~ 일 : 1 ~ 7 이다.

유닉스의 cron 은 일 ~ 일 : 0 ~ 7 이다. 그래서 ISO-8601 과도 호환이 된다. 유닉스 셸에서 `man 5 crontab`로 확인 가능.

그런데 Quartz 는 일 ~ 토 : 1 ~ 7 이다. [여기](https://github.com/quartz-scheduler/quartz/blob/d42fb7770f287afbf91f6629d90e7698761ad7d8/quartz-core/src/main/java/org/quartz/DateBuilder.java#L56) 참고

그래서 Quartz 안에서야 오류 없이 정해진 요일에 잘 실행되겠지만,
Quartz 의 스케줄에서 읽은 요일을 LocalDate 또는 cron 식의 값과 비교하거나,  
LocalDate 또는 cron 식에서 읽은 요일값을 그대로 Quartz 스케줄에 사용하면 의도와 다르게 동작하게 된다.

관련해서 2019.10.26 에 https://github.com/quartz-scheduler/quartz/issues/524 이런 이슈가 올라왔으나,  
2020.05 현재도 아직 응답은 없다.

Quartz 요일은 조심해서 쓰자.
