# 잘못 구현된 Redis Distributed Lock

다음과 같이 레디스 분산락을 사용해서 동시성 처리를 하고 있었다.

```java
    public long acquireLock(String lockKey, long expireMillis, long waitMillis) {
        long count = op.increment(lockKey, 1);
        if (count > 1) {
            try {
                long elapsed = 0;
                long sleep = 50;
                while (elapsed < waitMillis) {
                    Thread.sleep(sleep);
                    elapsed += sleep;
                    sleep = Math.min(sleep * 2, waitMillis - elapsed + 1);
                    count = op.increment(lockKey, 1);
                    if (count == 1) {
                        break;
                    }
                }
            } catch (InterruptedException ignored) {
            }

            if (count > 1) {
                throw new ErrorTypeBizException(ErrorType.COM_FAIL_TO_GET_LOCK);
            }
        }

        redisTemplate.expire(lockKey, expireMillis, TimeUnit.MILLISECONDS);
        return count;
    }

    public void releaseLock(String lockKey) {
        redisTemplate.delete(lockKey);
    }
```

몇 가지 문제가 있지만 실 운영 중에 눈에 띄게 발현되는 문제는 없었다고 한다.

문제가 뭔지 왜 눈에 띈 오류는 없었는지 하나씩 살펴보자.

## non-atomic

- op.increment(lockKey, 1)은 레디스에 lockKey 만료 시점 없는 엔트리를 생성한다.

- 이후 과정에서 에러가 발생해서 끝부분의 redisTemplate.expire()가 실행되지 않으면 엔트리가 레디스게 계속 남는 문제 발생


## lock 해제 문제

- lock 해제(레디스 엔트리 삭제) 관련하여 지금처럼 lockKey 하나에만 의존하는 방식은 동시성 문제 발생 환경에서
taskA가 생성한 lock 을 나중에 시작된 taskB의 finally 에서 먼저 해제할 수 있고, 이 경우 taskA 가 종료되기 전에 taskC 가 새로 lock 을 생성해서 taskA, taskC 실행이 겹칠 수 있음

- 반대로 taskA가 생성한 lock 이 taskA 작업을 마치기 전에 만료되면, taskB가 새로 lock 을 생성해서 taskA, taskB 실행이 겹칠 수 있고, taskA의 finally 에서 taskB가 생성한 lock 을 해제할 수 있음


## 2배 backoff 문제

- 위 구현부에 sleep = Math.min(sleep * 2, waitMillis - elapsed + 1);  와 같이 50ms 에서 시작해서 spin 주기를 2배씩 늘리는 로직이 있음

- lock 획득 대기 시간을 짧게 지정하면 문제가 없지만 5초로만 지정해도, 50+100+200+400+800+1600 까지 3.15초를 기다려도 lock 을 획득하지 못하면,

- 다음 순서에는 3200ms를 기다려야 하므로, 3.16초에 기존 lock 이 해제되어 lock을 가질 수 환경이 됐음에도 불구하고 3.2초를 기다리다가 5초 만료되어 lock을 획득하지 못하게 됨

- 동시 요청이 많을 수록, lock 을 획득할 수 있음에도 대기하다가 lock 획득을 못하는 상황이 더 많이 발생


## 실무적으로 이슈 발생 없던 이유

- non-atomic 문제로 만료 없는 캐시 엔트리(lock)가 생성되더라도, 다른 task가 생성한 lock 을 해제할 수 있는 문제점이 만료 없는 캐시 엔트리를 삭제할 수 있으므로, 두 단점이 서로 상쇄되면서 키 누적 문제 발생하지 않았을 것으로 추정

- 하지만 lock 이 해제되지 않아야 하는 상황에서도 lock이 다른 task에 의해 강제로 해제되어 다른 task가 critical section에 진입할 수 있으므로 코드의 의도와 다르게 동작하는 문제는 여전히 남음 

- 이로 인해 동시성 문제 발생 위험이 있더라도, taskA와 taskB가 어느 정도의 선후 관계가 있다면 실질적인 데이터 정합성 문제가 발생하지 않을 수 있음


## 대응

- 위 세 가지 문제를 보완한 RedisUUIDLockHandler 새로 추가

```java
    public @Nullable String acquireSpinLock(String lockKey, long expireMillis, long waitMillis) {
        if (StringUtils.isBlank(lockKey)) {
            return null;
        }

        String uuidLockValue = UUID.randomUUID().toString();
        ValueOperations<String, String> op = redisTemplate.opsForValue();
        if (Boolean.TRUE == op.setIfAbsent(lockKey, uuidLockValue, expireMillis, TimeUnit.MILLISECONDS)) {
            return uuidLockValue;
        }

        try {
            long elapsed = 0;
            long sleep = 100;
            while (elapsed < waitMillis) {
                Thread.sleep(sleep);
                elapsed += sleep;
                if (Boolean.TRUE == op.setIfAbsent(lockKey, uuidLockValue, expireMillis, TimeUnit.MILLISECONDS)) {
                    return uuidLockValue;
                }
            }
        } catch (InterruptedException ignored) {
        }

        throw new ErrorTypeBizException(ErrorType.COM_FAIL_TO_GET_LOCK);
    }

    public void releaseSpinLock(String lockKey, String uuidLockValue) {
        if (StringUtils.isBlank(lockKey) || StringUtils.isBlank(uuidLockValue)) {
            return;
        }

        String luaScript = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        DefaultRedisScript<Long> script = new DefaultRedisScript<>();
        script.setScriptText(luaScript);
        script.setResultType(Long.class);

        redisTemplate.execute(script, List.of(lockKey), uuidLockValue);
    }
```

- 0.1초 단위로 레디스에 요청을 보내게 되므로 5초 대기 기준으로 기존에 최대 6번만 호출하던 것에 비해 최대 50번을 호출하는 단점 존재

  - 하지만 실제로는 lock 획득 가능성이 높아지므로 대기하는 상황 자체가 줄어들어 여러 task에 의한 전체 호출 수는 오히려 줄어들 수도 있음

- lettuce, redisson, redlock 등 다른 안정된 구현체를 사용하는 것도 검토 필요
