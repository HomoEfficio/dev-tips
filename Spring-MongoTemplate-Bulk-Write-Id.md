# Spring MongoTemplate Bulk Write with Id

BulkOperations를 활용해서 Bulk 연산을 수행할 수 있는데, BulkOperations를 만드는 방법에 따라 Id 필드 이름도 다르게 지정해야한다.

특히 다음과 같이 **String인 `xxxCollectionName`을 사용해서 BulkOperations 객체를 만들었다면 Query에 사용하는 Id 필드 이름을 반드시 `_id`로 지정해야 하고 `id`로 지정하며 해당 다큐먼트를 찾지 못한다.**

```kotlin
val bulkOperationsXxx = mongoTemplate.bulkOps(BulkOperations.BulkMode.ORDERED, xxxCollectionName)  // xxxCollectionName는 String
        
bulkOperationsXxx.upsert(
    Query.query(
        Criteria().andOperator(
//                    Criteria.where("id").`is`(ObjectId(xxxId)),  // id로 하면 찾지 못함
            Criteria.where("_id").`is`(ObjectId(xxxId)),
        ),
    ),
    Update()
        .set(aaa, bbb)
)
```

하지만 Class를 사용해서 BulkOperations 객체를 만들었다면 Query에 사용하는 Id 필드 이름을 `id`로 지정해도 `_id`로 지정할 때와 마찬가지로 해당 다큐먼트를 정상적으로 찾는다.

```kotlin
val bulkOperationsXxx = mongoTemplate.bulkOps(BulkOperations.BulkMode.ORDERED, Xxx::class.java)  // 두 번째 인자가 class 
        
bulkOperationsXxx.upsert(
    Query.query(
        Criteria().andOperator(
//                    Criteria.where("id").`is`(ObjectId(xxxId)),  // id로 해도 찾을 수 있음
            Criteria.where("_id").`is`(ObjectId(xxxId)),
        ),
    ),
    Update()
        .set(aaa, bbb)
)
```

결국 **BulkOperations을 사용할 때 Id 컬럼 이름은 `_id`로 지정해야 문제가 없다.**
