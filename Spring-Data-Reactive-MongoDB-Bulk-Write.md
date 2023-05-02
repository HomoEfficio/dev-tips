# Spring Data Reactive MongoDB Bulk Write

건 수가 굉장히 많은 데이터를 한 건 한 건 쓰는 것에 비해 여러 건을 묶어서 쓰는 Bulk Write(insert/update/delete)는 성능 향상 효과가 굉장히 크다.

그래서 대부분의 데이터베이스에서는 이름은 조금씩 다르겠지만 Bulk Write 기능을 지원하며, 몽고디비도 물론 지원한다.

다만 Reactive MongoDB 대한 예제가 많지 않은데,  
아마도 Bulk Write를 하는 작업이라면 배치 작업일테고,  
배치 작업에 굳이 Reactive를 사용할 필요가 없기 때문 아닐까 싶기도 하지만,  
암튼 써야하는 상황이라면 써야하니 예제 하나 남겨둔다.

## bulkWrite

리액티브 몽고디비에서는 결국 아래와 같은 모양새로 Bulk Write를 실행할 수 있다.

```kotlin
val bulkWriteResult: Mono<BulkWriteResult> = reactiveMongoTemplate.getCollection("my_collection")
    .bulkWrite(List<UpdateOneModel<Document>>, BulkWriteOptions)
    .toMono()
return bulkWriteResult
```

핵심인 `MongoCollection.bulkWrite()` API는 [여기](https://mongodb.github.io/mongo-java-driver/4.8/apidocs/mongodb-driver-reactivestreams/com/mongodb/reactivestreams/client/MongoCollection.html#bulkWrite(java.util.List,com.mongodb.client.model.BulkWriteOptions))에 있고, 아래와 같다.

```
bulkWrite

Publisher<BulkWriteResult> bulkWrite(List<? extends WriteModel<? extends TDocument>> requests,
                                     BulkWriteOptions options)

Executes a mix of inserts, updates, replaces, and deletes.

Parameters:
requests - the writes to execute
options - the options to apply to the bulk write operation

Returns:
a publisher with a single element the BulkWriteResult
```

결국 묶어 쓰기할 데이터와 필요한 부가 정보를 어떻게 requests와 options로 구성해서 bulkWrite()을 호출할 것이냐로 귀결된다.

구구절절 설명 대신 예제 코드를 보는 게 낫다.

## 간단 시나리오

`my_collection`에 1 ~ 70 까지의 oid를 가지는 다큐먼트가 저장돼 있다고 하자.

```json
[
  { "oid": 1 },
  { "oid": 2 },
  // ...
  { "oid": 70 },
]
```

1 ~ 50까지는 delete하고, 51 ~ 100까지는 +1000 한 값을 upsert하는 작업을 bulkWrite()를 통해 진행해보자.

### requests

requests는 업데이트 할 데이터 요청이나 명세라고 생각하면 된다.

```kotlin
val requests: List<WriteModel<Document>> = IntStream.rangeClosed(1, 100).mapToObj { num ->
    when {
        num < 51 -> {
            DeleteOneModel<Document>(
                Filters.and(
                    BsonDocument("oid", BsonInt32(num))
                ),
            )
        }

        else -> {
            UpdateOneModel<Document>(
                Filters.and(
                    BsonDocument("oid", BsonInt32(num))
                ),
                Document("\$set", BsonDocument("oid", BsonInt32(num + 1000))),
                UpdateOptions().upsert(true),
            )
        }
    }
}.peek {
    log.info("$it")
}.toList()
```

삭제할 때는 `DeleteOneModel<Document>`를 사용한다. 삭제 대상 데이터 식별에 필요한 필터만 `Filters`로 구성해주면 된다.

수정할 때는 `UpdateOneModel<Document>`를 사용한다. 수정 대상 데이터 식별에 필요한 필터를 `Filters`로 구성해주고,  
set 할 데이터도 `Document`를 사용해서 지정해줘야 한다.
이 때 주의할 점은 set 할 다큐먼트를 인자로 넘겨주면 안 되고 `Document("\$set", set할다큐먼트)`와 같이 넘겨줘야 한다는 점이다.  
이렇게 해주지 않고 set 할 다큐먼트를 그대로 인자로 넘겨주면 `Invalid BSON field name` 에러가 발생한다.  
필터에 존재하지 않는 데이터는 새로 생성하도록 Upsert를 사용할 것이므로 옵션으로 `UpdateOptions().upsert(true)`를 지정해준다.

이렇게 하나의 requests에 delete/update/insert를 한 꺼번에 넣을 수 있다는 점도 기억해두자.

### options

BulkWrite options는 `BulkWriteOptions`를 이용해서 지정해주면 된다. 옵션이 많지는 않으며 예제에서는 간단하게 데이터 변경이 순서대로 일어나도록 지정해준다.

```kotlin
val bulkWriteOptions: BulkWriteOptions = BulkWriteOptions().ordered(true)
```

## 전체 코드

보기 편하게 private 메서드 없이 `run()` 메서드에 모두 넣은 전체 코드는 다음과 같다.

배치 작업이므로 편의상 `block()`을 사용한다.

```kotlin
@Component
@Profile("practice")
class BulkWritePracticeRunner(
    private val reactiveMongoTemplate: ReactiveMongoTemplate,
) : CommandLineRunner {

    override fun run(vararg args: String?) {
        bulkWriteExample().block()
    }

    @Transactional
    fun bulkWriteExample(): Mono<Boolean> {

        val stopWatch = StopWatch("BulkWrite")

        stopWatch.start("데이터 입력")
        val myCollectionName = "my_tmp_practice"
        val success: Success = reactiveMongoTemplate.getCollection(myCollectionName)
            .insertMany(
                IntStream.rangeClosed(1, 70).mapToObj { Document("oid", BsonInt32(it)) }
                    .peek { log.info("$it") }.toList()
            )
            .toMono()
            .block()
        stopWatch.stop()

        log.info("++++++++++")

        val requests: List<WriteModel<Document>> = IntStream.rangeClosed(1, 100).mapToObj { num ->
            when {
                num < 51 -> {
                    DeleteOneModel<Document>(
                        Filters.and(
                            BsonDocument("oid", BsonInt32(num))
                        ),
                    )
                }

                else -> {
                    UpdateOneModel<Document>(
                        Filters.and(
                            BsonDocument("oid", BsonInt32(num))
                        ),
                        Document("\$set", BsonDocument("oid", BsonInt32(num + 1000))),
                        UpdateOptions().upsert(true),
                    )
                }
            }
        }.peek {
            log.info("$it")
        }.toList()

        val bulkWriteOptions: BulkWriteOptions = BulkWriteOptions().ordered(true)

        log.info("++++++++++")


        stopWatch.start("bulkWrite")

        return reactiveMongoTemplate.getCollection(myCollectionName)
            .bulkWrite(requests, bulkWriteOptions)
            .toMono()
            .doOnNext { bulkResult ->
                val end = System.currentTimeMillis()
                log.info(
                    """
                    Bulk Write Result
                    Deleted : ${bulkResult.deletedCount}
                    Matched : ${bulkResult.matchedCount}
                    Updated : ${bulkResult.modifiedCount}
                    Inserted: ${bulkResult.insertedCount}
                    """.trimIndent()
                )
                stopWatch.stop()
                log.info(stopWatch.prettyPrint())
            }
            .thenReturn(true)
    }

    private val log = LoggerFactory.getLogger(javaClass)
}
```

## 결과

```
[           main] : Document{{oid=BsonInt32{value=1}}}
[           main] : Document{{oid=BsonInt32{value=2}}}
[           main] : Document{{oid=BsonInt32{value=3}}}
[           main] : Document{{oid=BsonInt32{value=4}}}
[           main] : Document{{oid=BsonInt32{value=5}}}
[           main] : Document{{oid=BsonInt32{value=6}}}
[           main] : Document{{oid=BsonInt32{value=7}}}
[           main] : Document{{oid=BsonInt32{value=8}}}
[           main] : Document{{oid=BsonInt32{value=9}}}
[           main] : Document{{oid=BsonInt32{value=10}}}
[           main] : Document{{oid=BsonInt32{value=11}}}
[           main] : Document{{oid=BsonInt32{value=12}}}
[           main] : Document{{oid=BsonInt32{value=13}}}
[           main] : Document{{oid=BsonInt32{value=14}}}
[           main] : Document{{oid=BsonInt32{value=15}}}
[           main] : Document{{oid=BsonInt32{value=16}}}
[           main] : Document{{oid=BsonInt32{value=17}}}
[           main] : Document{{oid=BsonInt32{value=18}}}
[           main] : Document{{oid=BsonInt32{value=19}}}
[           main] : Document{{oid=BsonInt32{value=20}}}
[           main] : Document{{oid=BsonInt32{value=21}}}
[           main] : Document{{oid=BsonInt32{value=22}}}
[           main] : Document{{oid=BsonInt32{value=23}}}
[           main] : Document{{oid=BsonInt32{value=24}}}
[           main] : Document{{oid=BsonInt32{value=25}}}
[           main] : Document{{oid=BsonInt32{value=26}}}
[           main] : Document{{oid=BsonInt32{value=27}}}
[           main] : Document{{oid=BsonInt32{value=28}}}
[           main] : Document{{oid=BsonInt32{value=29}}}
[           main] : Document{{oid=BsonInt32{value=30}}}
[           main] : Document{{oid=BsonInt32{value=31}}}
[           main] : Document{{oid=BsonInt32{value=32}}}
[           main] : Document{{oid=BsonInt32{value=33}}}
[           main] : Document{{oid=BsonInt32{value=34}}}
[           main] : Document{{oid=BsonInt32{value=35}}}
[           main] : Document{{oid=BsonInt32{value=36}}}
[           main] : Document{{oid=BsonInt32{value=37}}}
[           main] : Document{{oid=BsonInt32{value=38}}}
[           main] : Document{{oid=BsonInt32{value=39}}}
[           main] : Document{{oid=BsonInt32{value=40}}}
[           main] : Document{{oid=BsonInt32{value=41}}}
[           main] : Document{{oid=BsonInt32{value=42}}}
[           main] : Document{{oid=BsonInt32{value=43}}}
[           main] : Document{{oid=BsonInt32{value=44}}}
[           main] : Document{{oid=BsonInt32{value=45}}}
[           main] : Document{{oid=BsonInt32{value=46}}}
[           main] : Document{{oid=BsonInt32{value=47}}}
[           main] : Document{{oid=BsonInt32{value=48}}}
[           main] : Document{{oid=BsonInt32{value=49}}}
[           main] : Document{{oid=BsonInt32{value=50}}}
[           main] : Document{{oid=BsonInt32{value=51}}}
[           main] : Document{{oid=BsonInt32{value=52}}}
[           main] : Document{{oid=BsonInt32{value=53}}}
[           main] : Document{{oid=BsonInt32{value=54}}}
[           main] : Document{{oid=BsonInt32{value=55}}}
[           main] : Document{{oid=BsonInt32{value=56}}}
[           main] : Document{{oid=BsonInt32{value=57}}}
[           main] : Document{{oid=BsonInt32{value=58}}}
[           main] : Document{{oid=BsonInt32{value=59}}}
[           main] : Document{{oid=BsonInt32{value=60}}}
[           main] : Document{{oid=BsonInt32{value=61}}}
[           main] : Document{{oid=BsonInt32{value=62}}}
[           main] : Document{{oid=BsonInt32{value=63}}}
[           main] : Document{{oid=BsonInt32{value=64}}}
[           main] : Document{{oid=BsonInt32{value=65}}}
[           main] : Document{{oid=BsonInt32{value=66}}}
[           main] : Document{{oid=BsonInt32{value=67}}}
[           main] : Document{{oid=BsonInt32{value=68}}}
[           main] : Document{{oid=BsonInt32{value=69}}}
[           main] : Document{{oid=BsonInt32{value=70}}}
[           main] : ++++++++++
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 1}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 2}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 3}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 4}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 5}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 6}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 7}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 8}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 9}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 10}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 11}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 12}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 13}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 14}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 15}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 16}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 17}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 18}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 19}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 20}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 21}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 22}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 23}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 24}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 25}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 26}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 27}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 28}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 29}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 30}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 31}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 32}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 33}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 34}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 35}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 36}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 37}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 38}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 39}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 40}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 41}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 42}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 43}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 44}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 45}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 46}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 47}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 48}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 49}]}, options=DeleteOptions{collation=null}}
[           main] : DeleteOneModel{filter=And Filter{filters=[{"oid": 50}]}, options=DeleteOptions{collation=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 51}]}, update=Document{{$set={"oid": 1051}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 52}]}, update=Document{{$set={"oid": 1052}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 53}]}, update=Document{{$set={"oid": 1053}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 54}]}, update=Document{{$set={"oid": 1054}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 55}]}, update=Document{{$set={"oid": 1055}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 56}]}, update=Document{{$set={"oid": 1056}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 57}]}, update=Document{{$set={"oid": 1057}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 58}]}, update=Document{{$set={"oid": 1058}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 59}]}, update=Document{{$set={"oid": 1059}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 60}]}, update=Document{{$set={"oid": 1060}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 61}]}, update=Document{{$set={"oid": 1061}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 62}]}, update=Document{{$set={"oid": 1062}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 63}]}, update=Document{{$set={"oid": 1063}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 64}]}, update=Document{{$set={"oid": 1064}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 65}]}, update=Document{{$set={"oid": 1065}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 66}]}, update=Document{{$set={"oid": 1066}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 67}]}, update=Document{{$set={"oid": 1067}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 68}]}, update=Document{{$set={"oid": 1068}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 69}]}, update=Document{{$set={"oid": 1069}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 70}]}, update=Document{{$set={"oid": 1070}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 71}]}, update=Document{{$set={"oid": 1071}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 72}]}, update=Document{{$set={"oid": 1072}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 73}]}, update=Document{{$set={"oid": 1073}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 74}]}, update=Document{{$set={"oid": 1074}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 75}]}, update=Document{{$set={"oid": 1075}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 76}]}, update=Document{{$set={"oid": 1076}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 77}]}, update=Document{{$set={"oid": 1077}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 78}]}, update=Document{{$set={"oid": 1078}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 79}]}, update=Document{{$set={"oid": 1079}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 80}]}, update=Document{{$set={"oid": 1080}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 81}]}, update=Document{{$set={"oid": 1081}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 82}]}, update=Document{{$set={"oid": 1082}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 83}]}, update=Document{{$set={"oid": 1083}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 84}]}, update=Document{{$set={"oid": 1084}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 85}]}, update=Document{{$set={"oid": 1085}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 86}]}, update=Document{{$set={"oid": 1086}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 87}]}, update=Document{{$set={"oid": 1087}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 88}]}, update=Document{{$set={"oid": 1088}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 89}]}, update=Document{{$set={"oid": 1089}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 90}]}, update=Document{{$set={"oid": 1090}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 91}]}, update=Document{{$set={"oid": 1091}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 92}]}, update=Document{{$set={"oid": 1092}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 93}]}, update=Document{{$set={"oid": 1093}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 94}]}, update=Document{{$set={"oid": 1094}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 95}]}, update=Document{{$set={"oid": 1095}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 96}]}, update=Document{{$set={"oid": 1096}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 97}]}, update=Document{{$set={"oid": 1097}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 98}]}, update=Document{{$set={"oid": 1098}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 99}]}, update=Document{{$set={"oid": 1099}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : UpdateOneModel{filter=And Filter{filters=[{"oid": 100}]}, update=Document{{$set={"oid": 1100}}}, options=UpdateOptions{upsert=true, bypassDocumentValidation=null, collation=null, arrayFilters=null}}
[           main] : ++++++++++
[ntLoopGroup-2-2] : Bulk Write Result
Deleted : 50
Matched : 20
Updated : 20
Inserted: 0
[ntLoopGroup-2-2] : StopWatch 'BulkWrite': running time = 428336291 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
281946833  066%  컬렉션 삭제
081141083  019%  데이터 입력
065248375  015%  bulkWrite


Process finished with exit code 0

```

71 - 100 까지의 데이터는 Upsert 에 의해 insert 됐음이 실제 데이터에서 확인됨에도 불구하고 Inserted가 0으로 찍히는 점이 아쉬운데,  
다른 예제로 확인해보니 InsertOneModel이 아니라 UpdateOneModel을 사용했기 때문에 Inserted가 0으로 찍히는 것이다.

**(추가) Upsert 된 카운트도 `${bulkResult.upserts.size}`를 통해 확인할 수 있다.**


## 복잡 시나리오

아주 단순한 예제를 통해 Bulk Write 감을 잡았으니 이제 실전에 가까운 시나리오도 한 번 시도해보자.

외부로부터 다음과 같은 일자별 통계 데이터를 받아서,

```json
[
    {
        "date": "2022-12-07",
        "active_user": 4679,
        "new_user": 2918,
        "total_user": 7093,
        "avg_duration_time_per_user": 513.69,
        "map_id": "net.parabola.miniature",
        "duration_time": 2403558,
        "avg_duration_time_per_connect": 338.86
    },
    {
        "date": "2022-12-07",
        "active_user": 47343,
        "new_user": 19433,
        "total_user": 133757,
        "avg_duration_time_per_user": 1007.34,
        "map_id": "com.zzsh.hideAndSeek",
        "duration_time": 47690957,
        "avg_duration_time_per_connect": 356.54
    }
]
```

배치 작업을 통해 다음과 같이 저장하는 시나리오다.

```json
[
    {
        "_id": "any-uuid",
        "map_id": "net.parabola.miniature",
        "stat_date":
        {
            "$date": "2022-12-07T00:00:00.000Z"
        },
        "stat_name": "ACTIVE_USER",
        "stat_value": 4679
    },
    {
        "_id": "any-uuid",
        "map_id": "net.parabola.miniature",
        "stat_date":
        {
            "$date": "2022-12-07T00:00:00.000Z"
        },
        "stat_name": "NEW_USER",
        "stat_value": 2918
    },
    {
        "_id": "any-uuid",
        "map_id": "net.parabola.miniature",
        "stat_date":
        {
            "$date": "2022-12-07T00:00:00.000Z"
        },
        "stat_name": "TOTAL_USER",
        "stat_value": 7093
    },
    {
        "_id": "any-uuid",
        "map_id": "net.parabola.miniature",
        "stat_date":
        {
            "$date": "2022-12-07T00:00:00.000Z"
        },
        "stat_name": "AVG_DURATION_TIME_PER_USER",
        "stat_value": 513.69
    },
    {
        "_id": "any-uuid",
        "map_id": "net.parabola.miniature",
        "stat_date":
        {
            "$date": "2022-12-07T00:00:00.000Z"
        },
        "stat_name": "AVG_DURATION_TIME_PER_CONNECT",
        "stat_value": 338.86
    },
    {
        "_id": "any-uuid",
        "map_id": "net.parabola.miniature",
        "stat_date":
        {
            "$date": "2022-12-07T00:00:00.000Z"
        },
        "stat_name": "DURATION_TIME",
        "stat_value": 2403558
    },
    // .. 이하 생략
]
```

받은 다큐먼트 1건이 통계 종류별로 6개의 다큐먼트로 분리되어 저장된다.

저장할 데이터가 많으므로 100건씩 잘라서 Bulk Write를 적용한다.

### 통계 항목 Enum

```kotlin
enum class StatName {
    ACTIVE_USER,
    NEW_USER,
    DURATION_TIME,
    TOTAL_USER,
    AVG_DURATION_TIME_PER_CONNECT,
    AVG_DURATION_TIME_PER_USER,
}
```

### 받아온 데이터

```kotlin
data class ReceivedDailyStats(
    val date: String,
    val map_id: String,
    val active_user: Long,
    val new_user: Long,
    val total_user: Long,
    val avg_duration_time_per_user: Double,
    val avg_duration_time_per_connect: Double,
    val duration_time: Long,
) {

    fun getDoubleStatOf(statName: StatName): Double {
        return when (statName) {
            StatName.ACTIVE_USER -> active_user.toDouble()
            StatName.NEW_USER -> new_user.toDouble()
            StatName.TOTAL_USER -> total_user.toDouble()
            StatName.DURATION_TIME -> duration_time.toDouble()
            StatName.AVG_DURATION_TIME_PER_USER -> avg_duration_time_per_user
            StatName.AVG_DURATION_TIME_PER_CONNECT -> avg_duration_time_per_connect
        }
    }
}
```

### 외부 통계 API 호출 함수

```kotlin
interface StatsApiClient {
    fun getDailyStats(startDate: String, endDate: String): Flux<ReceivedDailyStats>    
}
```


### 저장할 데이터

```kotlin
@Document(collection = "daily_stats")
@CompoundIndexes(
    CompoundIndex(def = "{'map_id': 1, 'stat_name': 1, 'stat_date': 1}", unique = true)
)
class DailyStat(

    @Field("map_id")
    var mapId: String,

    @Field("stat_name")
    var statName: StatName,

    @Field("stat_date")
    var statDate: Instant,

    @Field("stat_value")
    var statValue: Double,

) : Serializable {

    fun toDocument(): BsonDocument {
        return BsonDocument()
            .append("map_id", BsonString(mapId))
            .append("stat_name", BsonString(statName.name))
            .append("stat_date", BsonDateTime(statDate.toEpochMilli()))
            .append("stat_value", BsonDouble(statValue))
    }
```

### Bulk Write 함수

```kotlin
override fun bulkSaveOrUpdate(dailyStats: List<DailyStat>): Mono<BulkWriteResult> {

    log.info("DailyStat BulkWrite")
    val bulkWriteOptions: BulkWriteOptions = BulkWriteOptions().ordered(true)

    val updateOneModels: List<UpdateOneModel<Document>> = dailyStats
        .map { dailyStat ->
            UpdateOneModel<Document>(
                Filters.and(
                    BsonDocument(
                        "project", BsonString(dailyStat.project)
                    ),
                    BsonDocument(
                        "stat_name", BsonString(dailyStat.statName.name)
                    ),
                    BsonDocument(
                        "stat_date", BsonDateTime(dailyStat.statDate.toEpochMilli())
                    ),
                ),
                Document("\$set", dailyStat.toDocument()),  // $set 안 해주고 그냥 dailyStat.toDocument()를 사용하면 Invalid BSON field name 에러 발생
                UpdateOptions().upsert(true),
            )
        }

    return reactiveMongoTemplate.getCollection("daily_stats")
        .bulkWrite(updateOneModels, bulkWriteOptions)
        .toMono()
}

```

### 100개씩 묶어 BulkWrite를 호출하는 함수

```kotlin
// 주입받은 의존 관계
// statsApiClient.getDailyStats()
// dailyStatsRepository.bulkSaveOrUpdate()

override fun saveDailyStats(fromDate: LocalDate, toDate: LocalDate): Mono<Boolean> {

    return statsApiClient.getDailyStats(
        startDate = fromDate.format(DateTimeFormatter.ISO_LOCAL_DATE),
        endDate = toDate.format(DateTimeFormatter.ISO_LOCAL_DATE)
    )  // Flux<ReceivedDailyStats> 반환
        .map { dailyStats: ReceivedDailyStats ->
            StatName.values()
                .map { statName ->
                    DailyStat(
                        statDate = LocalDate.parse(dailyStat.date).toUTCInstant(),
                        project = dailyStats.map_id,
                        statName = statName,
                        statValue = dailyStats.getLongStatOf(statName)
                    )
                }
        }  // Flux<ReceivedDailyStats> -> Flux<List<DailyStat>> 반환
        .flatMap { it: List<DailyStat> ->
            Flux.fromIterable(it)
        }  // Flux<List<DailyStat>> -> Flux<DailyStat> 반환
        .bufferTimeout(100, Duration.ofSeconds(1))  // Flux<DailyStat> -> 100개씩 묶어서 Flux<List<DailyStat>> 반환. bulkWrite()이 List를 인자로 받으므로 window 사용하면 안 되고 buffer 사용해야 함
        .flatMap { it: List<DailytStat> ->. // 100개씩 묶어놓은 List
            dailyStatsRepository.bulkSaveOrUpdate(it)
        }  // Flux<List<DailyStat>> -> Flux<BulkWrite> 반환
        .doOnNext { bulkResult: BulkWriteResult ->
            log.info(
                """
                Matched : ${bulkResult.matchedCount}
                Inserted: ${bulkResult.insertedCount}
                Updated : ${bulkResult.modifiedCount}
                """.trimIndent()
            )
        }
        .then()
        .thenReturn(true)
```

### 결과

실제 저장된 데이터가 234건이고 100개씩 묶어서 Bulk Write 했으니 아래와 같이 3번 실행되는 것까지는 확인이 되는데,  
앞서 살펴본 것처럼 UpdateOneModel을 사용했기 때문에 Inserted는 0이라고 나오는 점이 아쉽다.

**(추가) `${bulkResult.upserts.size}`를 통해 Upsert 된 카운트도 확인할 수 있다.**

```
[ctor-http-nio-1] : DailyStat BulkWrite
[ctor-http-nio-1] : DailyStat BulkWrite
[ctor-http-nio-1] : DailyStat BulkWrite
[ntLoopGroup-2-4] : Matched : 0
Inserted: 0
Updated : 0
[ntLoopGroup-2-2] : Matched : 0
Inserted: 0
Updated : 0
[ntLoopGroup-2-3] : Matched : 0
Inserted: 0
Updated : 0
```

## 한 걸음 더

### 저장 순서

InsertOneModel로 바꿔서 실해해보면 결과가 다음과 같이 Inserted 카운트가 표시된다.

```
[ctor-http-nio-1] : DailyStat BulkWrite
[ctor-http-nio-1] : DailyStat BulkWrite
[ctor-http-nio-1] : DailyStat BulkWrite
[ntLoopGroup-2-4] : Matched : 0
Inserted: 34
Updated : 0
[ntLoopGroup-2-2] : Matched : 0
Inserted: 100
Updated : 0
[ntLoopGroup-2-3] : Matched : 0
Inserted: 100
Updated : 0
```

그런데 가장 마지막에 저장돼야 할 것 같은 34개가 가장 먼저 저장되네??  
생각해보면 당연하기도 하다.
3개의 서로 다른 스레드에서 벙렬 실행 되는데,  
100개 저장보다 34개 저장이 시간이 덜 걸릴테니 34개 저장이 먼저 완료되는 것이다.

이러면 의도와 다르게 저장 순서가 달라지는 거 아닌가 싶은데,  
실제 생성된 데이터를 `_id` 값으로 내림차순 정렬해보면 의도했던 순서대로 표시되는 것을 확인할 수 있다.

문서상으로 확인해보지는 않았지만 결과로 추정해볼 때 `_id`값과 timestamp는 저장 완료 시점에 계산되는 것이 아니라 저장이 시작될 때 정해지는 것으로 보인다.

재미있는 점은 timestamp 기준으로 보면 나머지 200개도 100개씩 동일한 timestamp를 가지고 있지 않고, 27개, 73개, 27개 73개로 나눠져 있다는 점인데 이 이유까지는 모르겠다.

3번의 Bulk Write를 위와 같이 각각 별도의 스레드 말고 모두 동일한 스레드에서 실행하고 싶다면 어떻게 해야할까?  
이렇게 하면 된다.

```kotlin
            .bufferTimeout(100, Duration.ofSeconds(1))  // bulkWrite()이 List를 인자로 받으므로 window 사용하면 안 되고 buffer 사용해야 함
            .flatMap { it ->
                dailyProjectStatRepository.bulkSaveOrUpdate(it)
                    .publishOn(Schedulers.single())  //// 여기에 단일 스레드 지정!!
            }
```

결과는 다음과 같이 모두 'single-1' 스레드에서 Bulk Write가 실행된다.

```
[ctor-http-nio-1] : DailyStat BulkWrite
[ctor-http-nio-1] : DailyStat BulkWrite
[ctor-http-nio-1] : DailyStat BulkWrite
[       single-1] : Matched : 0
Inserted: 34
Updated : 0
[       single-1] : Matched : 0
Inserted: 100
Updated : 0
[       single-1] : Matched : 0
Inserted: 100
Updated : 0
```

timestamp 기준으로 보면 이번에는 100, 27, 73, 34로 묶여있다.

그런데 **순서를 보장하는 가장 좋은 방법은 flatMap 대신 concatMap 을 사용**하는 것이다.

```kotlin
            .bufferTimeout(100, Duration.ofSeconds(1))  // bulkWrite()이 List를 인자로 받으므로 window 사용하면 안 되고 buffer 사용해야 함
            .concatMap { it ->  ////// flatMap 대신 concatMap!!
                dailyProjectStatRepository.bulkSaveOrUpdate(it)
                    .publishOn(Schedulers.single())  //// 여기에 단일 스레드 지정!!
            }
```

### BackPressure

리액티브가 진정한 가치를 뽐내는 지점은 복잡하고 현란한 리액티브 연산자가 아니라 **Consumer 자신이 스스로 처리할 수 있을 만큼만 Producer 쪽에 요청할 수 있는 BackPressure**다.

위 예제에 BackPressure를 적용하려면 어떻게 해야하까?  
여러가지 방법이 있겠지만 `limitRate()`를 사용하는 방법이 가장 직관적이다.

```kotlin
    return statsApiClient.getDailyStats(
        startDate = fromDate.format(DateTimeFormatter.ISO_LOCAL_DATE),
        endDate = toDate.format(DateTimeFormatter.ISO_LOCAL_DATE)
    )  // Flux<ReceivedDailyStats> 반환
        .limitRate(50)  //// 여기!! - 최대 50개 들어있는 Flux<ReceivedDailyStat> 반환
        .map { dailyStats: ReceivedDailyStats ->
            StatName.values()
                .map { statName ->
                    DailyStat(
                        statDate = LocalDate.parse(dailyStat.date).toUTCInstant(),
                        project = dailyStats.map_id,
                        statName = statName,
                        statValue = dailyStats.getLongStatOf(statName)
                    )
                }
        }  // Flux<ReceivedDailyStats> -> Flux<List<DailyStat>> 반환
        .flatMap { it: List<DailyStat> ->
            Flux.fromIterable(it)
        }  // Flux<List<DailyStat>> -> Flux<DailyStat> 반환
        .bufferTimeout(100, Duration.ofSeconds(1))  // Flux<DailyStat> -> 100개씩 묶어서 Flux<List<DailyStat>> 반환. bulkWrite()이 List를 인자로 받으므로 window 사용하면 안 되고 buffer 사용해야 함
```

이렇게 하면 최대 50개씩만 조회 요청을 보내서 50개씩만 받은 후에 이후 로직을 처리한다.
