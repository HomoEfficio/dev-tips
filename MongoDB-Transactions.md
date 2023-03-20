# MongoDB Transaction

## Storage Engine

- https://www.mongodb.com/docs/v4.4/core/wiredtiger/

- 3.2부터 WiredTiger 스토리지 엔진

### WiredTiger 스토리지 엔진

- Document level Concurrency 지원
- 글로벌, 데이터베이스, 컬렉션 수준의 RW에서 Optimisic Concurrency Control([intent lock](https://www.mongodb.com/docs/v4.4/reference/glossary/#std-term-intent-lock)) 사용
  - 두 개의 쓰기 중 충돌 발생 시 하나만 W되고 나머지 하나는 실패 후 몽고디비가 알아서 Retry?
- MVCC(Multi Version Concurrency Control)
  - 연산 시작 시 point-in-time snapshot 이 연산에 제공된다
    - 스냅삿은 인메모리 데이터의 consistent view를 보여준다
  - 연산에 의한 저장되는 데이터는 스냅샷에 반영되고, 스냅샷에 있는 데이터는 모든 데이터 파일에 걸쳐 일관성을 유지한채 디스크에 저장된다
  - 쉽게 말해 **Tx1 시작 시 해당 시점의 데이터 스냅샷이 Tx1에 제공**되고,
    - **Tx2 시작 시 해당 시점의 데이터 스냅샷이 Tx2에 제공**된다.
    - 따라서 **Tx1에서 아직 커밋되지 않은 데이터는 Tx2에 보이지 않으며, 마찬가지로 Tx2에서 아직 커밋되지 않은 데이터는 Tx1에 보이지 않는다.**
    - **Tx1 에서 변경하는 데이터와 동일한 데이터를 Tx2에서 변경하려고 하면 WriteConflict 발생**
      - **Tx1 에서 변경하는 데이터와 Tx2 에서 변경하는 데이터가 동일한 컬렉션에 있더라도 다른 다큐먼트면 Conflict 발생 안함**
  - 방금 디스크에 쓰여진 데이터(정확하게는 다수결 충족하는 노드 숫자만큼의 저널 파일에 쓰여진 데이터)는 데이터 파일의 체크포인트로 동작
    - 3.6부터 60초 단위 또는 2G 한도로 체크포인트 생성
    - 체크포인트는 일종의 데이터 지속성(durability) 기본 보장 단위로서 장애나면 최종 체크포인트까지의 데이터 복원 가능
      - Journaling을 사용하면 체크포인트 이후 ~ 서버 사망 시점의 데이터도 복원 가능
  - 새 체크포인트를 참조하도록 와이어드타이거의 메타데이터 테이블이 원자성을 유지한 채 업데이트되면 새 체크포인트에 액세스할 수 있고 영구적으로 유지. 새 체크포인트에 액세스할 수 있게 되면 WiredTiger는 이전 체크포인트에서 페이지를 해제
  

### Journal

- **데이터 지속성 보장을 위해 체크포인트와 함께 Write-Ahead-Log(WAL 또는 Journal) 사용**
  - 데이터 변경을 반영하기 전에 디스크 같은 지속성 있는 매체에 로그를 먼저 기록
- **와이어드타이거 저널은 체크포인트 사이의 데이터 변경을 모두 저장하는 체계 또는 저장하는 파일(노드별로 존재)을 의미**
- 저널 사용 시 데이터 복원 과정
  1. 데이터 파일에서 마지막 체크포인트의 식별자 A 탐색
  2. 저널 파일에서 A에 해당하는 레코드 탐색
  3. 저널 파일에서 마지막 체크포인트 이후에 해당하는 레코드(연산)를 찾아 DB에 적용
- 데이터 수정이 발생하고 이로 인해 인덱스 수정도 발생한다면, 저널 파일 하나의 레코드에 데이터 수정 연산과 인덱스 수정 연산 모두를 기록
- 저널 파일에 저장되기 전 128kb의 인메모리 버퍼에 먼저 저장
- 저널링 데이터에는 snappy 압축 적용
  - 인덱스에는 prefix compression 적용
- 저널 파일은 최대 약 100MB, 초과하면 새 저널 파일 생성
- 오래된 저널 파일은 와이어드타이거가 자동으로 삭제


## 트랜잭션

- https://www.mongodb.com/docs/v4.4/core/transactions/

- 몽고디비에서 단일 다큐먼트에 대한 연산은 원자적(all-or-nothing)
  - 몽고디비는 중첩 다큐먼트와 배열을 사용해서 데이터 사이의 관계를 단일 다큐먼트 안에 내포할 수 있으므로, 멀티-다큐먼트 트랜잭션에 대한 필요가 적다.
- 분산(distributed) 트랜잭션을 사용하면 여러 연산 사이, 여러 컬렉션 사이, 여러 데이터베이스 사이, 여러 도큐먼트 사이, 여러 샤드 사이에서 트랜잭션 관리 보장

>중요
>- 리플리카셋과 샤드 클러스터에 대한 Tx가 보장되는 4.2에서는 4.2용 드라이버 사용 필수
>- 트랜잭션 안에 있는 각 연산에는 세션이 전달돼야 한다
>- 트랜잭션 안에 있는 각 연산은 트랜잭션 수준에서 정해진 read concern, write concern, read preference를 사용한다
>- 4.2와 그 이전버전에서는 트랜잭션 안에서 컬렉션 생성 불가
>  - 트랜잭션 안에서의 쓰기는 이미 존재하는 컬렉션에서만 가능
>- 4.4부터는 트랜잭션 안에서 컬렉션 생성 가능


## 트랜잭션과 원자성

- https://www.mongodb.com/docs/v4.4/core/transactions/#transactions-and-atomicity

>노트
>- 4.0에서는 리플리카셋에서의 멀티다큐 트랜잭션만 보장
>- 4.2에서 도입된 분산 트랜잭션은 4.0의 리플리카셋 멀티다큐 트랜잭션 보장에 추가로 샤드 클러스터에서의 멀티다큐 트랜잭션도 보장
>- 따라서 **4.2부터 분산 트랜잭션(distributed tx)과 멀티다큐 트랜잭션(multi-document tx)는 사실상 동의어**+다.

- 멀티다큐 트랜잭션은 원자적
- 트랜잭션이 커밋되기 전까지 트랜잭션 안에서 수행된 데이터 변경은 트랜잭션 외부로 노출되지 않는다.
- 트랜잭션 안에서의 쓰기가 여러 샤드에 걸쳐 수행되는 경우, 트랜잭션 외부에서 수행되는 모든 읽기 작업이 트랜잭션이 커밋되고 모든 샤드에 걸쳐 노출될 때까지 기다릴 필요는 없다(낙관적).
  - 트랜잭션이 커밋되고 트랜잭션 안에 있던 `쓰기1` 결과가 샤드A에서 노출되었지만 같은 트랜잭션 안에 있던 `쓰기2` 결과가 아직 샤드B에서 노출되지 않았을 때,
  - read concern이 'local'인 트랜잭션 외부의 읽기 작업은 `쓰기1`의 결과는 볼 수 있고, `쓰기2`의 결과는 볼 수 없을 뿐, `쓰기2` 결과 노출이 완료될 때까지 `쓰기1` 결과도 볼 수 없는 것은 아니다.
- 트랜잭션이 중단(abort)되면 트랜잭션 안에서 발생한 모든 변경은 어디에도 노출되지 않고 버려진다.

>중요
>- **멀티다큐 트랜잭션은 거의 대부분의 상황에서 단일다큐 쓰기에 비해 훨씬 더 많은 비용이 든다.**
>- 따라서 **멀티다큐 트랜잭션이 가능하다고 해서 스키마 디자인을 소홀히 해서는 안 된다.**
>  - 생각보다 많은 상황에서 반(비/탈)정규화(denormalization) 데이터 모델이 적합하며 이를 통해 멀티다큐 트랜잭션 수요를 최소화 할 수 있다.
>- 트랜잭션 사용 관련 실행 시간 제한이나 oplog 크기 제한 등은 [Production Considerations](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/)를 참고한다.

>팁
>- [Outside Reads During Commit](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/#std-label-transactions-prod-consideration-outside-reads)


## 트랜잭션과 연산

- https://www.mongodb.com/docs/v6.0/core/transactions/#transactions-and-operations

- 분산 트랜잭션은 여러 연산, 여러 컬렉션, 여러 데이터베이스, 여러 다큐먼트, (4.2부터)여러 샤드에 걸쳐 사용될 수 있다.
- 4.4부터 트랜잭션 안에서 컬렉션과 인덱스 생성 가능하다.
- 하나의 트랜잭션 안에서 서로 다른 데이터베이스에 존재하는 컬렉션의 데이터 연산도 처리할 수 있다.

### 트랜잭션에서 수행할 수 있는 연산

- (4.4부터)컬렉션과 인덱스 생성(제약은 있음)
- CRUD
- count
- distinct
- hello, buildInfo, connectionStatus 같은 정보 확인성 연산(트랜잭션의 첫 번째 연산이 아닐 때만)

### 트랜잭션에서 수행할 수 없는 연산

#### Capped Collection 관련

- capped collection은 high-throughput 연산을 지원하는 고정 크기 컬렉션
- 4.2부터 capped collection에 write 불가
- 5.0부터 capped collection으로부터 데이터를 읽을 때 Read Concern을 snapshot으로 지정할 수 없다.

#### 기타 불가 연산

- config, admin, local 데이터베이스에 있는 컬렉션에 대한 읽기/쓰기 연산
- system.* 컬렉션에 쓰기 불가
- 지원되는 연산에 대한 explain 같은 query plan연산 불가
- 트랜잭션 밖에서 생성된 커서는 트랜잭션 안에 있는 getMore 호출 불가
- 트랜잭선 안에서 생성된 커서는 트랜잭션 밖에 있는 getMore 호출 불가
- (4.2부터) 트랜잭션의 첫 번째 연산으로 killCursors 연산 불가


## 트랜잭션과 세션

- https://www.mongodb.com/docs/v4.4/core/transactions/#transactions-and-sessions
- 하나의 트랜잭션은 하나의 세션과 연관된다.
- 트랜잭션이 커밋되지 않은 채 연관된 세션이 종료되면 트랜잭션은 버려진다.


## 트랜잭션과 Read Preference

- https://www.mongodb.com/docs/v4.4/core/transactions/#transactions-and-read-preference
- 트랜잭션을 시작할 때 트랜잭션 수준의 Read Preference 지정 가능
  - 지정하지 않으면 세션 수준의 Read Prefeence 가 사용되고 그마저도 없으면 클라이언트 수준 Read Preference(기본값 primary) 사용
- 멀티다큐 트랜잭션에서 읽기 연산을 수행하려면 Read Preference가 반드시 primary여야하며, 트랜잭션 내 모든 연산은 동일한 멤버로 라우팅돼야 한다.

### Read Preference

- **레플리카셋 멤버 중 누구로부터 데이터를 읽어올지 지정**

read preference mode | 설명
--- | ---
primary (기본값) | 레플리카셋의 primary 멤버로부터 읽으며, primary 멤버가 다운이면 에러 발생
primaryPreferred | primary 멤버로부터 읽을 수 없는 경우 secondary 멤버로부터 읽는다
secondary | 레플리카셋의 secondary 멤버로부터 읽는다
secondaryPreferred | secondary 멤버들이 모두 다운된 경우 primary 멤버로부터 읽는다
nearest | primary, secondary와 무관하게 latency threshold 기준으로 지연 시간이 임계치 이내에 있는 멤버 중 임의의 멤버로부터 읽는다

- secondary 멤버의 staleness 계산
  - primary가 있다면 primary의 마지막 쓰기 시간과 각 secondary 마지막 쓰기 시간과의 차이
  - primary가 없다면 secondary 중에서 가장 최근에 쓰여진 secondary의 마지막 쓰기 시간과 각 secondary 마지막 쓰기 시간과의 차이


## 트랜잭션과 Read Concern

- https://www.mongodb.com/docs/v4.4/core/transactions/#transactions-and-read-concern
- 트랜잭션 수준 read concern은 트랜잭션 시작 시 지정 가능
  - 트랜잭션 수준 read concern이 지정돼 있지 않으면 세션 수준 read concern이 사용
  - 세션 수준 read concern이 지정돼 있지 않으면 클라이언트 수준 read concern(기본값 local)이 사용

### local

- 읽기 연산이 실행된 노드의 최신 데이터 반환, 반환된 이후에 롤백될 수도 있음
- 샤드 간 트랜잭션에서는 샤드간 동일한 스냅샷 뷰가 반환되는 것을 보장하지 않음

### majority

- 다수결 커밋된 데이터 반환
- w: majority 로 커밋된 경우 읽은 데이터가 롤백 되지 않음을 보장
  - w: majority 가 아닌(w: 1) write concern으로 커밋된 경우 읽은 데이터가 롤백될 수도 있음

### snapshot

- w: majority 로 커밋된 경우에 한해 다수결 커밋된 데이터의 스냅샷 반환
- 샤드 간 트랜잭션에서 데이터의 snapshot 뷰는 샤드간에 동기화 된다


## 트랜잭션과 Write Concern

- https://www.mongodb.com/docs/v4.4/core/transactions/#transactions-and-write-concern
- 트랜잭션 수준 write concern은 트랜잭션 시작 시 지정 가능
  - 트랜잭션 수준 write concern이 지정돼 있지 않으면 세션 수준 write concern이 사용
  - 세션 수준 write concern이 지정돼 있지 않으면 클라이언트 수준 write concern(기본값 w: 1)이 사용
- 트랜잭션 안에 있는 쓰기 연산은 지정된 트랜잭션 수준의 write concern을 사용한다
  - 연산 단위로 명시적인 write concern 지정을 할 수 없고 지정돼 있는 write concern으로 커밋 된다
    - 책 20.2.1, 20.2.2 에 나오는 write concern 지정 연산은 트랜잭션 안에서의 연산이 아님

### w: 1

- primary 멤버에만 커밋되면 커밋된 것으로 간주(returns acknowledgement)
  - 트랜잭션 수준 read concern이 majority 이면, 읽기 연산이 다수결 커밋된 데이터를 읽는다고 보장할 수 없음
  - 트랜잭션 수준 read concern이 snapshot 이면, 읽기 연산이 다수결 커밋된 스냅샷 데이터를 읽는다고 보장할 수 없음
  - **primary 멤버에만 커밋되고 어떤 레플리카 셋에도 복제되지 않은 상태에서 primary 멤버가 죽으면 해당 쓰기 연산은 롤백될 수 있다.**

### w: majority

- 다수결 충족 가능한 수의 멤버에 커밋돼야 커밋된 것으로 간주(returns acknowledgement)
  - 트랜잭션 수준 read concern이 majority 이면, 읽기 연산이 다수결 커밋된 데이터를 읽는다고 보장
    - 샤드 클러스터에 걸친 트랜잭션인 경우 다수결 커밋 데이터와 샤드와 동기화되지는 않음
  - 트랜잭션 수준 read concern이 snapshot 이면, 읽기 연산이 다수결 커밋되고 동기화 된 스냅샷 데이터를 읽는다고 보장



# 관련 주요 개념

## Read Preference

- 레플리카셋 멤버 중 누구로부터 데이터를 읽어올지 지정

read preference mode | 설명
--- | ---
primary (기본값) | 레플리카셋의 primary 멤버로부터 읽으며, primary 멤버가 다운이면 에러 발생
primaryPreferred | primary 멤버로부터 읽을 수 없는 경우 secondary 멤버로부터 읽는다
secondary | 레플리카셋의 secondary 멤버로부터 읽는다
secondaryPreferred | secondary 멤버들이 모두 다운된 경우 primary 멤버로부터 읽는다
nearest | primary, secondary와 무관하게 latency threshold 기준으로 지연 시간이 임계치 이내에 있는 멤버 중 임의의 멤버로부터 읽는다

- secondary 멤버의 staleness 계산
  - primary가 있다면 primary의 마지막 쓰기 시간과 각 secondary 마지막 쓰기 시간과의 차이
  - primary가 없다면 secondary 중에서 가장 최근에 쓰여진 secondary의 마지막 쓰기 시간과 각 secondary 마지막 쓰기 시간과의 차이

### primary

- primary 멤버에서 실행된 연산은 secondary 멤버에서 비동기로 실행되므로, primary 모드가 아니라면 최신화가 되지 않은 데이터를 읽을 가능성이 있다.
- Read Preference는 Read Isolation이나 Causal Consistency에 영향을 미치지 않는다.
- primary 모드는 tag set lists나 maxStalenessSeconds를 지정하면 에러 발생한다.
- 트랜잭션 안에서 발생하는 모든 연산은 동일한 멤버에서 실행돼야 하는 읽기 연산이 포함된 멀티다큐 트랜잭션은 항상 primary 모드에서만 가능

### primaryPreferred

- primary 멤버가 다운된 경우
  - secondary 멤버 중에서 가장 최신 상테인 멤버의 추정 지연 시간이 
    - maxStalenessSeconds보다 작으면 그 secondary 멤버로부터 데이터를 읽고,
    - maxStalenessSeconds보다 크면
      - tag set과 가장 먼저 매칭되는 secondary 멤버로부터 데이터를 읽고,
        - 매칭되는 secondary 멤버가 없으면 읽기 에러 발생
- 4.4부터 샤드 클러스터 환경에서는 primaryPreferred 모드에서 [hedged reads](https://www.mongodb.com/docs/manual/core/sharded-cluster-query-router/#std-label-mongos-hedged-reads) 지원


### secondary

- secondary 멤버 중에서 가장 최신 상테인 멤버의 추정 지연 시간이 
  - maxStalenessSeconds보다 작으면 그 secondary 멤버로부터 데이터를 읽고,
  - maxStalenessSeconds보다 크면
    - tag set과 가장 먼저 매칭되는 secondary 멤버로부터 데이터를 읽고,
      - 매칭되는 secondary 멤버가 없으면 읽기 에러 발생

### secondaryPreferred

- secondary 멤버 중에서 가장 최신 상테인 멤버의 추정 지연 시간이 
  - maxStalenessSeconds보다 작으면 그 secondary 멤버로부터 데이터를 읽고,
  - maxStalenessSeconds보다 크면
    - tag set과 가장 먼저 매칭되는 secondary 멤버로부터 데이터를 읽고,
      - 매칭되는 secondary 멤버가 없으면 primary 멤버로부터 데이터를 읽는다
- 4.4부터 샤드 클러스터 환경에서는 secondaryPreferred 모드에서 [hedged reads](https://www.mongodb.com/docs/manual/core/sharded-cluster-query-router/#std-label-mongos-hedged-reads) 지원

### nearest

- 지연 시간이 임계치 안에 있는 레플리카 셋 멤버 중에서 primary, secondary 무관하게 임의의 멤버로부터 데이터를 읽는다
- 데이터의 최신성보다 읽기 연산에 대한 네트워크 지연 영향 최소화가 더 중요할 때 사용
- 추정 지연 시간이 maxStalenessSeconds보다 큰 secondary는 제외
- tag set list가 있으면 tag set list에 없는 멤버는 제외
- 네트워크 지연 시간이 localThresholdMS 보다 큰 멤버는 제외
- 제외되지 않고 남은 멤버 중에서 primary, secondary 무관하게 임의의 멤버로부터 데이터를 읽는다
- 4.4부터 샤드 클러스터 환경에서는 nearest 모드에서 [hedged reads](https://www.mongodb.com/docs/manual/core/sharded-cluster-query-router/#std-label-mongos-hedged-reads) 지원


### 기타

- $out, $merge 스테이지를 포함하는 애그리게이션 파이프라인은 Read Preference 설정과 무관하게 무조건 primary 멤버에서 실행된다.


## Read Concern

- https://www.mongodb.com/docs/v4.4/reference/read-concern/#read-concern
- 레플리카 셋이나 레플리카 셋 샤드로부터 읽어온 데이터의 consistency, isolation 제어
- Read Concern과 Write Concern을 잘 조합하면 consistency, availability를 필요에 맞게 적용 가능
  - 일반적으로 일관성을 높이려면 가용성이 떨어지고, 가용성을 높이면 일관성이 떨어진다.
- 4.4부터 레플리카셋이나 샤드 클러스터에서 글로벌 read concern 기본값 지정 가능

### local

- 레플리카 셋 다수결을 획득하지 못한 데이터를 반환하며, 이후 다수결에 따라 해당 데이터가 롤백될 수도 있다.
- primary 멤버로부터 데이터를 읽는 게 기본이고 causally consistent session이면 secondary로부터 읽는다
  - causally consistent client 세션은 연관된 읽기 연산의 read concern이 majority이고 write concern도 majority일 때만 causal consistency 보장

### available

- 레플리카 셋 다수결을 획득하지 못한 데이터를 반환하며, 이후 다수결에 따라 해당 데이터가 롤백될 수도 있다.
- 기본: causally consistent session이 아니면 secondary로부터 읽는다.
- 사용가능 환경: causally consistent session에서는 사용 불가
- 샤드 클러스터에서는 Read Concern 중에서 available이 응답 지연이 가장 낮음
  - 단 orphaned documents를 반환할 수도 있음
    - 이를 방지하려면 local 사용

### majority

- 레플리카 셋 다수결을 획득한 데이터만 반환하며, 반환된 데이터는 durable하다.
- 레플리카 셋 멤버는 majority-commit 시점에서 데이터의 인메모리 뷰에 있는 데이터를 반환한다.
- 다른 read concern에 비해 성능 상으로는 불리
- causally consistent session이든 아니든 모두 사용 가능
- 레플리카 셋이 WiredTiger 스토리지 엔진일 때만 사용 가능
- 멀티다큐 트랜잭션에서는 Write Concern이 majority일 때만 다수결 획득 데이터만 반환하는 것이 보장

### linearizable

- 읽기 연산 시작 전에 다수결 획득을 완료한 데이터를 반환한다. 쿼리는 동시에 실행된 write가 다수결을 획득할 때까지 기다릴 수도 있다.(앞 문장과 충돌)
- writeConcernMajorityJournalDefault 가 true로 설정돼 있으면 읽기 연산 시작 후에 레플리카 셋 멤버가 죽더라도 읽기 연산에 사용된 데이터는 durable하다.
  - false이면 w: majority 쓰기가 완료될 때까지 기다리지 않는다.
- linearizable로 설정되면 $out, $merge stage 사용 불가
- 사용가능 환경: causally consistent session에서는 사용 불가
- 요구사항: 쿼리 필터가 단일 다큐를 반환하도록 구성됐을 때만 linearizable 보장

### snapshot

- 


## Write Concern

- https://www.mongodb.com/docs/v4.4/reference/write-concern/#write-concern

## Causal Consistency

- https://www.mongodb.com/docs/v4.4/core/read-isolation-consistency-recency/#causal-consistency
- 어떤 작업이 논리적으로 이전 작업에 의존하는 경우 작업 간에 인과 관계(causal relationship)이 존재
  - 예를 들어, 지정된 조건에 따라 모든 문서를 삭제하는 쓰기 작업과, 삭제 작업을 확인하는 후속 읽기 작업은 인과 관계 있음
- Causal Consistency 가 적용된 세션에서는 인과 관계를 깨지 않는 순서로 연산을 실행한다




