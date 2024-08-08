# Mongo DB Shard Key

## 샤드키란

- 데이터를 여러 샤드에 걸쳐 분산 저장 시, 분산의 기준이 되는 키
- 컬렉션의 인덱스 된 하나 또는 그 이상의 필드를 샤드키로 지정할 수 있다.
  - 해시 샤딩: `sh.shardCollection( "database.collection", { <field> : "hashed" } )`
  - 레인지 샤딩: `sh.shardCollection( "database.collection", { <field> : 1 } )`

## 샤드키 선정

다음 특성을 고려해서 적절한 필드를 샤드키로 선정해야 한다

- Large cardinality: 값의 가짓수가 많아야 여러 샤드로 분산할 수 있다. 단 이것만으로 균등 분산을 보장하지는 않음
  - ![](https://www.mongodb.com/docs/v4.2/_images/sharded-cluster-ranged-distribution-low-cardinal.bakedsvg.svg)
  - cardinality가 낮다면 cardinality가 높은 필드와 복합인덱스를 구성해서 샤드키에 사용
- Low frequency: 값의 출현 빈도수가 적어야 데이터의 쏠림이 적으므로 Hot이 덜 발생한다
  - ![](https://www.mongodb.com/docs/v4.2/_images/sharded-cluster-ranged-distribution-frequency.bakedsvg.svg)
- Monotonicity: 값이 단조증가/감소하면 쓰기 집중 발생
  - ![](https://www.mongodb.com/docs/v4.2/_images/sharded-cluster-monotonic-distribution.bakedsvg.svg)

## 제약

- 배열값이 있는 필드는 (여러 값 중 어느 값을 기준으로 분산 저장할 지 알 수 없으므로) 샤드키에 사용 불가
- 컬렉션의 샤드키를 한 번 선정하면 변경 불가, 4.2부터 샤드키에 사용되는 필드가 `_id`가 아니라면 샤드키의 값은 변경 가능
- 비어있지 않은 컬렉션을 샤드할 때는
  - 샤드키로 사용된 단일 컬럼에 대한 인덱스
  - 또는 샤드키로 시작하는 복합 인덱스가 컬렉션에 반드시 포함돼 있어야 함
  - All sharded collections must have an index that supports the shard key; i.e. the index can be an index on the shard key or a compound index where the shard key is a prefix of the index.
- 해시 인덱스에는 unique 지정 불가


## 종류

### Hashed Shard Key

- 샤드키로 지정된 필드 값의 해시값이 샤드키 값
  - ObjectId나 timestamp처럼 단조(monotonically)증가값을 가진 필드를 지정하는 것이 이상적
- 부동소수점 수는 버림처리되어 정수로 간주된 후에 해시되므로 1과 1.999의 해시값이 동일(동일한 청크에 저장됨)
- 좋은 해시 함수를 사용한다면 쓰기 분할이 잘 되어 Hot이 발생하지 않음
- point(equal, in) query 시 Target 쿼리로 실행, range query 실행 시 Broadcast 쿼리로 실행됨
- ![컬렉션에 해시 샤딩 적용](https://s7280.pcdn.co/wp-content/uploads/2020/11/Creating-the-sharding-dataset-3.png)

### Ranged Shard Key(책에는 없음)

- 샤드키로 지정된 필드 값을 기준으로 (가능한) 균등한 크기를 갖도록 범위를 나눠서 저장
- Non monotonicity 필요
  - 샤드키 값이 단조증가 하면 새 데이터는 계속 하나의 샤드에만 집중적으로 insert되어 Hot 발생


### Ranged vs Hashed

- Ranged
  - ![](https://www.mongodb.com/docs/v4.2/_images/sharded-cluster-monotonic-distribution.bakedsvg.svg)
  - 장점: range query가 target operation으로 실행될 수 있음
  - 단점: 단조증가 필드(ex: `_id`)를 샤드키로 사용 시 Hot 발생

- Hashed
  - ![](https://www.mongodb.com/docs/v4.2/_images/sharded-cluster-hashed-distribution.bakedsvg.svg)
  - 장점: 단조증가 필드(ex: `_id`)를 샤드키로 사용해도 Hot 발생 안함
  - 단점: range query가 broadcast operation으로 실행됨

## 기타

### GridFS 샤딩

- https://www.mongodb.com/docs/v4.2/core/gridfs/#sharding-gridfs
- chunks 컬렉션에는 샤딩 적용이 의미 있음
  - 기본적으로 `{ files_id : 1, n : 1 } or { files_id : 1 }`를 샤드키로 사용하는 레인지 샤딩
  - 파일 업로드 성공 확인을 위해 filemd5 를 실행하지 않는 몽고디비 드라이버(몽고디비 4.0 이상 지원하는 드라이버) 사용 시 해시. 샤딩도 가능
  - filemd5 를 실행하는 몽고디비 드라이버 사용 시 해시 샤딩 불가능
- files 컬렉션에는 파일 메타 정보만 있으므로 샤딩이 사실 상 무의미하며 샤딩하지 않고 primary 샤드에 두는 것을 권장
  - 정 샤딩해야겠다면 `_id` 필드를 샤드키로 사용(애플리케이션 필드랑 혼합 사용도 가능)

### 파이어호스(매뉴얼에서 못 찾음)

- 여러 서버 중 성능 좋은 서버에 더 많은 부하가 걸리게 하자
  - 성능 좋은 서버를 넘어서는 부하가 들어오면 부하를 나눌 방법이 없다능..

### 다중 핫스팟(매뉴얼에서 못 찾음)

- 뭔소린지 모르겠고 적절히 구성한 복합인덱스를 샤드키로 사용해서 샤드별로 몇몇 핫스팟 청크가 발생하게 해서 부하를 나누자는 얘기인 듯


## 샤딩을 사용 중인데도 특아헌 현황

- 샤딩 사용 중이긴 한데 Sharded Collection은 없다!?
  - `db.printShardingStatus()` 명령을 실행하면 이상한 에러

    ```
    studio> db.printShardingStatus()
            ;
    [2023-02-02 14:22:01] Command failed with error 59 (CommandNotFound): 'no such cmd: hello' on server 10.105.143.235:3718. The full response is {"ok": 0.0, "errmsg": "no such cmd: hello", "code": 59, "codeName": "CommandNotFound", "operationTime": {"$timestamp": {"t": 1675315321, "i": 9}}, "$clusterTime": {"clusterTime": {"$timestamp": {"t": 1675315321, "i": 9}}, "signature": {"hash": {"$binary": {"base64": "m/M9LqbEVE7B42vppVmET0iR+Bs=", "subType": "00"}}, "keyId": 7163540785806180366}}}
  
    ```
  - `db.printCollectionStats()` 명령으로 아래 내용 확인
    ```
    db.printCollectionStats()
  
    [
      {
        "admin_users": {
          "sharded": false,
          ...
        },
        "assets": {
          "sharded": false,
          ...
        },
        ...
      }
    ]
    결과 중 "sharded": true 인 컬렉션이 하나도 없음
    ```
  - `db.collection.getShardDistribution()` 로 샤드 여부 확인 가능
    ```
    db.items.getShardDistribution()
    ;
  
    MongoshInvalidInputError: [SHAPI-10001] Collection items is not sharded at Proxy.getShardDistribution (all-standalone.js:6282:21219) at async Proxy.getShardDistribution (all-standalone.js:6295:8239) at async Proxy.r (all-standalone.js:6295:2648) at async Proxy.r (all-standalone.js:6295:31 ... 
    ```


