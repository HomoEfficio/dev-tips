# Spring PlatformTransactionManager Commit 누락 시 DB Lock 해제

Spring을 사용하면 `@Transactional`을 통해 편리하게 Transaction을 관리할 수 있다.  
그런데 `@Transactional`을 사용하기 어려운 상황도 있는데, 이 때는 `PlatformTransactionManager`를 사용해서 다음과 같이 수동으로 Tx를 설정할 수 있다.

```java
@Autowired
private PlatformTransactionManager transactionManager;

...

    TransactionStatus transactionStatus = transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
        // 어쩌구 저장
    } catch (Exception e) {
        transactionManager.rollback(transactionStatus);
        throw new XXXException(e);
    }
    transactionManager.commit(transactionStatus);
  
...
```

그런데 **실수로 commit 을 잊고 누락했다면 어떻게 될까?**  
결과는 심각하다. 일반적으로 row 기반 Lock을 사용하므로 **Tx 시작 이후에 동일 스레드에서 수행된 저장에 관련된 모든 데이터의 레코드에 Lock이 걸리고 풀리지 않게 된다.**

이 Lock을 풀려면 두 가지 방법이 있다.

- DBMS에서 해당 Lock을 찾아서 직접 해제
- 해당 Lock을 일으킨 서버 인스턴스를 내리고 DB와의 연결을 끊어서 롤백 유도

어느 방법이든 Lock이 해제되면서 롤백 된다.
