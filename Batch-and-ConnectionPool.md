# 배치 작업과 커넥션풀

## 커넥션풀

일반적으로 DB에 연결해서 어떤 작업을 할 때는 커넥션풀(Connection Pool)을 사용한다. DB 연결 자체가 비용이 많이 들기 때문에 미리 다수의 Connection 객체를 만들어서 풀에 넣어두고 필요할 때마다 꺼내쓰고 반납하기를 반복한다. 결국 응답 속도를 빠르게 하고 자원 효율성을 높이기 위해 커넥션풀을 사용한다.

커넥션풀을 사용하면 미리 만들어진 연결을 여러 곳에서 재사용하기 때문에 연결 객체에 어떤 공통의 상태를 두고 이를 변경해가면서 사용하면 안 된다. A라는 작업에서 필요에 의해 그 상태를 변경하면, A가 쓰고 반납한 연결 객체를 재사용하는 B라는 다른 작업에서는 A에 의해 변경된 상태에 의해 의도하지 않게 동작할 위험이 있기 때문이다.


## 배치 작업

하지만 배치(bach) 작업은 어떨까? 대부분의 배치 작업은 최종 사용자의 요청과는 무관하게 동작하며 따라서 최종 사용자에게 필요한 수준의 응답 속도를 필요로 하지 않는다. 그리고 보통 작업량도 많아서 DB 연결 생성 비용은 그 작업에 필요한 전체 비용과 비교하면 무시할 정도로 미미한 수준이다. 따라서 **일반적인 배치 작업에서는 커넥션풀의 효용이 그다지 크지 않다.**

게다가 작업 성격 상 커넥션별로 어떤 설정값을 변경해야 하는 경우도 많다. 예를 들어 Hive의 [Dynamic Partition Insert](https://github.com/HomoEfficio/dev-tips/blob/master/Hive%20Dynamic%20Partition%20Insert.md)을 사용할 때는 아래 `set hive.exec.XXX=YYY`와 같이 설정값을 변경해줘야 한다.

![Imgur](https://i.imgur.com/gTXD7Sp.png)

**a, b로 설정한 게 c 실행 시 까지 유효해야 하는데, 실제로는 a, b, c 모두 서로 다른 커넥션에서 실행된다.** 이는 크게 세 가지 치명적인 문제를 유발한다.

1. a, b 설정 내용은 해당 커넥션에 그대로 남아서 나중에 다른 작업에 영향을 미친다. 다른 작업은 의도한 것과 다르게 동작할 수 있다는 얘기다.
1. a, b 설정 내용은 c 실행 시에는 적용되지 않는다. 셋 모두 별개의 커넥션에서 실행되기 때문이다.
1. 1, 2에 의한 문제는 간헐적, 우발적으로 발생한다. 재연이 어렵고 디버깅이 어렵다.

참고로 a, b, c 실행 시 서로 다른 커넥션이 사용된다는 건 아래의 JdbcTemplate 구현 내용에서 확인할 수 있다.

![Imgur](https://i.imgur.com/A3LyoRc.png)

위와 같이 결국 JdbcTemplate의 DB 작업 메서드가 실행될 때마다 `DataSourceUtil.getConnection()`로 커넥션을 가져오는데, 이게 결국 커넥션풀에서 그때그때 커넥션을 새로 가져온다.


## 배치 작업에서의 커넥션풀

앞에서 얘기한 것처럼 배치 작업에서의 커넥션풀의 효용은 크지 않다. 그리고 커넥션별 설정 변경 같은 사용 사례가 필요한 상황에서는 커넥션풀을 써서 커넥션을 재사용하는 것은 앞에서 얘기한 것처럼 해결이 어려운 문제를 유발하는 부작용만 떠안을 뿐이다.

그럼 커넥션풀을 안 쓰면 되는 거 아닌가? 맞다. 근데 실무적으로 편하게 사용할 수 있는 JdbcTemplate이 커넥션풀에서 커넥션을 가져오게 되어 있으므로 이 편리함을 그대로 유지하려면 커넥션풀을 사용해야 한다. 뭐야 그럼 어쩌라고?

커넥션풀을 사용하되 적당히 설정하면 커넥션을 사용할 때마다 커넥션 객체를 새로 생성하게 할 수 있다.

즉 커넥션풀을 사용하지만 일반적인 커넥션풀처럼 커넥션을 미리 만들어두고 재사용하는 게 아니라, 

- 커넥션을 사용할 때마다 늘 새로 만들고,
- 사용 후에 바로 폐기되며,
- 동시 사용 커넥션의 최대 갯수만 제한을 두는

특별한 풀을 만들 수 있다.

아래는 Tomcat의 PoolProperties를 사용할 때의 설정값인데 다른 ConnectionPool 구현체를 쓰더라도 비슷한 설정방법이 있을 것이다.

```yml
  driverClassName: a.b.c.d.GoodDriver
  url: JDBC_URL
  username: USERNAME
  password: PASSWORD
  initialSize: 0  # 풀 생성 시 커넥션 객체를 미리 생성하지 않음
  maxActive: 100  # 동시 사용 가능 한 커넥션 객체의 갯수 상한값을 기본값인 100으로 명시
  maxIdle: 0      # 사용 후 대기 중인 커넥션의 최대 갯수를 0으로 고정
  minIdle: 0      # 사용 후 대기 중인 커넥션의 최소 갯수를 0으로 고정
  minEvictableIdleTimeMillis: 0  # 사용 후 폐기 전 대기 시간을 0밀리초로 고정 -> 사용 후 바로 폐기
```

## Tx 처리

위와 같이 커넥션풀을 설정하면 앞에서 살펴본 1번 문제는 해결할 수 있다. 하지만 a, b 설정이 c 실행시까지 유지되어야 하는 2번 문제는 어떻게 해결해야 할까?

하나의 Tx로 묶어주면 JdbcTemplate이 처음에 가져온 커넥션을 Tx 종료시까지 계속 사용할 수 있다. 스프링이라면 `@Transactional`을 사용하거나 아래와 같이 PlatformTransactionManager 를 사용해서 하나의 Tx로 묶어줄 수 있다.

```java
TransactionStatus transactionStatus = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

JdbcTemplate hiveTezJdbcTemplate = hiveTezDataReader.createJdbcTemplate();

// 아래 3개의 명령이 모두 동일한 커넥션 객체를 통해 실행됨
hiveTezJdbcTemplate.execute("set hive.exec.dynamic.partition.mode=nonstrict");
hiveTezJdbcTemplate.execute("set hive.exec.max.dynamic.partitions=50000");
hiveTezJdbcTemplate.execute(hiveCustomerConsumptionQuery);

this.transactionManager.commit(transactionStatus);

```






