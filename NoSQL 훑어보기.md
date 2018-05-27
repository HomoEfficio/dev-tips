# NoSQL 훑어보기

원형섭 님의 'NoSQL 이론' 강의를 바탕으로 이것저것 주워모아서 붙이고 떼고 정리

## 등장 배경

- 모바일, SNS 등의 발전으로 데이터량의 폭발적 증가
  - 특히 비정형 데이터(로그 등)의 양과 중요성 증가
- 대규모 비정형 데이터 처리에 대한 RDB의 한계
  - Scale Out 한계
    - 자동 샤딩을 대부분 지원하지 않음
  - 정형화 된 스키마
    - 비정형 데이터 처리에 부적합
  - 고비용
    - 대량 로그 처리를 위해 값비싼 Oracle RAC를 쓸 것인가
  - 대규모 비정형 데이터
    - 수백 테라, 페타바이트의 데이터를 RDB에서 SQL로 정렬, 집계는 사실상 불가능
  - RDB에 이런 한계가 있는 이유는 RDB가 나온 시점에는 일관성이 무엇보다 중요한 요소였기 때문

## 주요 특징

RDB의 한계의 대칭

- 분산 DB를 바탕 개념으로 깔고 Scale Out 지원
- 스키마 불필요
- (대부분)오픈소스

>분산과 유연성 확보를 위해 일관성을 양보

## 바탕 이론

### BASE

> Basically Available, Soft state, Eventual Consistency

- 기본적 가용성: 부분적 장애가 발생할 수는 있어도 시스템 전체가 중단되지는 않음
- 소프트한 상태: 데이터 사본의 엄격한 일관성을 요구하지 않으며 노드의 상태는 내부 데이터가 아닌 외부로부터의 데이터(ex: replication)에 의해 결정
- 결과적 일관성: 특정 시점에는 일관성이 깨져 있을 수 있지만 어떤 인위적인 개입이 없더라도 시간이 지나면 종국적으로(결과적으로)는 일관성이 맞춰짐

>RDB에서 보장하는 ACID와 상반되는 개념

### CAP

> 분산 시스템에서 Consistency, Availability, Partition Tolerance 중 3가지 모두를 달성할 수는 없다.

- A: 분산 시스템의 특정 노드에 장애가 발생하더라도 클라이언트는 분산 시스템에 데이터를 RW 할 수 있다.
  - 특정 요청이 장애로 인해 분산 시스템 전체에서 처리되지 못한다면 A 훼손
- P: 노드 사이에 전송되는 메시지가 네트워크에서 손실될 수 있다.
  - 즉, P를 배제하려면 메시지가 손실될 수 없는 네트워크를 구성해야 하는데, 이는 현실적으로 불가능

>결국 CAP 중에서 P는 무조건 선택되어야 하고, C와 A 중에서 하나만을 선택할 수 있다.

>참고: http://eincs.com/2013/07/misleading-and-truth-of-cap-theorem/

### PACELC

![](https://i.stack.imgur.com/lV2pB.png)

> Partition 발생(네트워크 장애)하면 Availability와 Consistency 중 하나만 선택 가능하고,
>- 네트워크 장애 시 데이터를 모든 노드에 전파할 수는 없으므로 접근 가능한 노드에만 데이터를 써서 A를 취하고 C를 포기하거나,
>- 또는 모든 노드에 전파되지 않으면 일부 노드에 전파된 데이터를 롤백 시켜서 C를 취하고 A를 포기해야 한다.
>
> Else이면(네트워크 장애 아니면) Latency와 Consistency 중 하나만 선택 가능하다.
>- 모든 노드에 쓰려면 지연 시간이 발생하므로 C는 취할 수 있지만 L은 포기해야 하고,
>
>- 반대로 일부 노드에만 써서 L을 취하고 C를 포기할 수도 있다.

## 주요 데이터 모델

모든 주요 데이터 모델이 아래 기술한 특성을 띤다기보다는 대부분의 데이터 모델이 아래 특성을 따른다

## Key-Value

![https://medium.baqend.com/nosql-databases-a-survey-and-decision-guidance-ea7823a822d](https://cdn-images-1.medium.com/max/1200/1*swUK-eLWsk-wudXSXRgyYQ.png)

- Value의 타입에 대해 신경쓰지 않으므로(opaque) 여러가지 타입(text, binary, list, set, map, ...)의 데이터를 저장할 수 있다(heterogeneous).
- Key를 Hash한 값을 통해 접근하므로 정렬에 적합하지 않고 결국 Range 쿼리에 적합하지 않다.
- Redis, DynamoDB, Riak 등

## Document

![https://medium.baqend.com/nosql-databases-a-survey-and-decision-guidance-ea7823a822d](https://cdn-images-1.medium.com/max/1200/1*gdxUo2ojiTX2JQIkA2hxcQ.png)

- 쉽게 말해 JSON을 하나의 Document라고 하고, Document 단위로 통째로 저장할 수 있는 데이터 모델
- Document 안에 다른 Document를 내장(embed)할 수 있으며, 내장 Document의 필드에도 secondary index를 설정할 수 있다.
- MongoDB, CouchDB 등

## Column Family

![https://medium.baqend.com/nosql-databases-a-survey-and-decision-guidance-ea7823a822d](https://cdn-images-1.medium.com/max/1600/1*jvxgAtqWADL8E43gs0p8jw.png)



## Graph

![https://whatis.techtarget.com/definition/graph-database](https://itknowledgeexchange.techtarget.com/overheard/files/2014/01/Graph-database-sketch.jpg)

- 엔티티(node)와 엔티티 사이의 관계(edge)를 저장
- NoSQL이지만 ACID와 일관성 보장

## 분산 모델

### Sharding

![https://www.cubrid.org/manual/ko/9.3.0/shard.html](https://www.cubrid.org/manual/ko/9.3.0/_images/image39.png)

샤딩은 동일한 스키마를 유지하면서 수평 방향으로 분할선을 그어 행을 기준으로 데이터를 분할하여 분산 DB에 저장하는 것을 말한다.

순차적 샤딩키를 적용하면 특정 노드에 쓰기가 집중되는 Hot Spot이 생길 수 있고, 추후 확장으로 인해 분할 구간 재구성이 어려울 수 있다.

비순차적(해시, modulo 등) 샤딩키를 적용하면 순차적 샤딩키의 단점을 극복할 수 있지만, 샤당키와 rowkey를 동일하게 써야하는 데이터베이스에서는 키 값에 의한 정렬이 안 되므로 range 쿼리를 사용할 수 없게 된다.


### Replication


## 키 모델

순차적 키를 사용하는 HBase와 해쉬 키를 사용하는 Cassandra는 모두 NoSQL이라는 분류표를 달고 있지만 키(Key) 모델에 따라 동작 방식이 판이하게 다르다. 

### 순차적 키

분산 DB가 아닌 단일 노드 DB에서는 자동 증가하는 순차적 키를 사용해도 큰 문제가 없다. 어차피 애초부터 하나의 노드에 쓰기가 집중되는 환경이고, 키 발급도 하나의 노드에서 이루어지므로 순서 꼬임 걱정도 없다. 오히려 순차적 키 덕분에 range 쿼리를 사용할 수 있으므로 순차적 키의 장점을 잘 활용할 수 있다.

하지만 분산 DB에서는 쓰기 집중이 발생하는 HotSpot, 키의 전파 등이 필요하므로 순차적 키의 단점이 부각된다.

>순차적 키는 정렬이 가능해서 range 쿼리를 사용할 수 있지만,
>
>분산 DB에서는 HotSpot, 키 순서 보장 등의 단점이 부각된다.

- HBase


### 비순차적 키

해시 등 비순차적 키의 경우 정렬이 불가능하므로 range 쿼리가 불가능하고 equal 쿼리만 가능하다는 단점이 있지만, 분산 DB에서는 HotSpot이 발생하지 않고 키 순서 보장이 필요 없어서 쓰기 성능이 좋아진다는 장점이 부각된다.

>비순차적 키는 range 쿼리가 불가능하고 equal 쿼리만 가능하지만,
>
>분산 DB에서는 HotSpot 미발생, 키 순서 보장 불필요 등의 장점이 부각된다.

- Redis, Cassandra
