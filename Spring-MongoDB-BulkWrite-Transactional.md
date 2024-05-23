# Spring MongoDB BulkWrite 와 Transactional

MongoDB의 BulkWrite에 의해 수행되는 Batch 연산은 기본적으로 Atomicity가 보장되지 않는다.

>https://www.mongodb.com/docs/manual/core/bulk-write-operations/
>
>With an ordered list of operations, MongoDB executes the operations serially. If an error occurs during the processing of one of the write operations, MongoDB will return without processing any remaining write operations in the list. See ordered Bulk Write
>
>With an unordered list of operations, MongoDB can execute the operations in parallel, but this behavior is not guaranteed. If an error occurs during the processing of one of the write operations, MongoDB will continue to process remaining write operations in the list. See Unordered Bulk Write Example.
>

쉽게 말하면 아래와 같은 상황에서

```
// 아래 3개의 연산이 하나의 Batch에 추가도어 BulkWrite 된다고 할 때
bulkOperations.operation1()
bulkOperations.operation2()  // 이 연산 수행 중 에러 발생!!
bulkOperations.operation3()

bulkOperations.execute()
```

Ordered BulkWrite 이면 operation1은 롤백되지 않고 operation3은 실행되지 않는다.  
Unordered BulkWrite 이면 operation1, operation3 모두 실행되고 모두 롤백되지 않는다.

Ordered/Unordered 일 때 동작이 살짝 다르기는 하지만 결국 **MongoDB의 BulkWrite에 의해 수행되는 Batch 연산은 기본적으로 Atomicity가 보장되지 않는다**는 점은 동일하다.

이럴 때 `@Transactional`이 출동하면 어떨까?

결론부터 말하면 다음과 같다.

>MongoDB의 BulkWrite에 의해 수행되는 Batch 연산은 기본적으로 Atomicity가 보장되지 않지만,  
>`@Transactional`이 붙어서 Tx가 적용되는 메서드 안에서 실행된 BulkWrite에 의해 수행되는 Batch 연산은 Atomicity가 보장된다.

쉽게 말해 다음과 같이 동작한다.

```kotlin
@Transactional
fun mongoBulkWrite() {
    // 아래 3개의 연산이 하나의 Batch에 추가도어 BulkWrite 된다고 할 때
    bulkOperations.operation1()
    bulkOperations.operation2()  // 이 연산 수행 중 에러 발생!!
    bulkOperations.operation3()
    
    bulkOperations.execute()
}
```

Ordered BulkWrite 이면 operation1은 실행되지만 롤백되고, operation3은 실행되지 않는다.  
Unordered BulkWrite 이면 operation1, operation3 모두 실행되지만 모두 롤백된다.

결국 operation1, 2, 3 을 하기 전과 동일한 상태로 되돌릴 수 있으므로,  
`@Transactional`의 도움을 받으면 BulkWrite 의 atomicity 도 보장할 수 있다.

물론 이 글에서는 `@Transactional`과 BulkWrite의 동작만 살펴봤을 뿐 성능 관점에서 살펴보지는 않았으니,  
실무에 사용하기 전에 실제 환경에서 확인 후 의도에 맞게 Transactional BulkWrite 를 사용하는 것이 좋다.

테스트 해본 코드는 대략 다음과 같다.

```kotlin
import com.mongodb.bulk.BulkWriteResult
import org.slf4j.LoggerFactory
import org.springframework.boot.CommandLineRunner
import org.springframework.context.annotation.Profile
import org.springframework.data.mongodb.core.BulkOperations
import org.springframework.data.mongodb.core.MongoTemplate
import org.springframework.data.mongodb.core.index.CompoundIndex
import org.springframework.data.mongodb.core.index.CompoundIndexes
import org.springframework.data.mongodb.core.mapping.Document
import org.springframework.data.mongodb.core.query.Criteria
import org.springframework.data.mongodb.core.query.Query
import org.springframework.stereotype.Component
import org.springframework.transaction.annotation.Transactional

@Component
@Profile("mongo-tx-text")
class MongoTxTestRunner(
    private val mongoTemplate: MongoTemplate,
    private val transactionalBatchProcessor: TransactionalBatchProcessor,
) : CommandLineRunner {

    override fun run(vararg args: String?) {

        orderedBulkWriteWithoutTransactional()

        unOrderedBulkWriteWithoutTransactional()

        transactionalBatchProcessor.orderedBatchInsert()

        transactionalBatchProcessor.unOrderedBatchInsert()
    }

    private fun setup() {
        mongoTemplate.findAllAndRemove(Query.query(Criteria.where("name").`is`(dupDocument().name)), MyTxTestDocument::class.java)
        mongoTemplate.insert(dupDocument())
    }

    private fun verify(bulkOperations: BulkOperations, expected: Int) {
        try {
            bulkOperations.execute()
        } catch (t: Throwable) {
            t.printStackTrace()
        } finally {
            val all = mongoTemplate.findAll(MyTxTestDocument::class.java)
            check(all.size == expected) {
                log.error("예상된 다큐먼트 수 $expected != 실제 다큐먼트 수 ${all.size}")
            }
        }
    }

    private fun dupDocument(): MyTxTestDocument = MyTxTestDocument(name = "abc", desc = "def")

    private fun unOrderedBulkWriteWithoutTransactional() {
        setup()

        log.info("unOrderedBatchInsert 시작")
        val unOrderedBulkOps = mongoTemplate.bulkOps(BulkOperations.BulkMode.UNORDERED, MyTxTestDocument::class.java)
        (1..15).forEach { num ->
            unOrderedBulkOps.insert(
                MyTxTestDocument(
                    name = "${num}th",
                    desc = "${num}th desc"
                )
            )  // UNORDERED 이면 모두 insert 됨
        }
        unOrderedBulkOps.insert(dupDocument())  // 미리 insert 돼 있던 다큐먼트
        (16..30).forEach { num ->
            unOrderedBulkOps.insert(
                MyTxTestDocument(
                    name = "${num}th",
                    desc = "${num}th desc"
                )
            )  // UNORDERED 이면 모두 insert 됨
        }

        verify(unOrderedBulkOps, expected = 31)
    }

    private fun orderedBulkWriteWithoutTransactional() {
        setup()

        log.info("orderedBatchInsert 시작")
        val orderedBulkOps = mongoTemplate.bulkOps(BulkOperations.BulkMode.ORDERED, MyTxTestDocument::class.java)
        orderedBulkOps.insert(MyTxTestDocument(name = "1st", desc = "1st desc"))  // insert 됨
        orderedBulkOps.insert(MyTxTestDocument(name = "2nd", desc = "2nd desc"))  // insert 됨
        orderedBulkOps.insert(dupDocument())  // 미리 insert 돼 있던 다큐먼트
        orderedBulkOps.insert(MyTxTestDocument(name = "3rd", desc = "3rd desc"))  // insert 안됨
        orderedBulkOps.insert(MyTxTestDocument(name = "4th", desc = "4th desc"))  // insert 안됨

        verify(orderedBulkOps, expected = 3)
    }

    private fun printBulkWriteResult(bulkWriteResult: BulkWriteResult) {
        log.info("bulkWriteResult.insertedCount: ${bulkWriteResult.insertedCount}")
        log.info("bulkWriteResult.deletedCount: ${bulkWriteResult.deletedCount}")
        log.info("bulkWriteResult.matchedCount: ${bulkWriteResult.matchedCount}")
        log.info("bulkWriteResult.modifiedCount: ${bulkWriteResult.modifiedCount}")
    }

    private val log = LoggerFactory.getLogger(javaClass)
}

@Document(collection = "my-tx-test")
@CompoundIndexes(
    CompoundIndex(def = "{'name': 1}", unique = true),
)
data class MyTxTestDocument(
    val id: String? = null,
    val name: String,
    val desc: String,
)

@Component
class TransactionalBatchProcessor(
    private val mongoTemplate: MongoTemplate,
) {

    @Transactional(transactionManager = "mongoTransactionManager")
    fun orderedBatchInsert() {
        setup()

        log.info("Transactional orderedBatchInsert 시작")
        val orderedBulkOps = mongoTemplate.bulkOps(BulkOperations.BulkMode.ORDERED, MyTxTestDocument::class.java)
        orderedBulkOps.insert(MyTxTestDocument(name = "1st", desc = "1st desc"))  // 롤백 됨
        orderedBulkOps.insert(MyTxTestDocument(name = "2nd", desc = "2nd desc"))  // 롤백 됨
        orderedBulkOps.insert(dupDocument())  // 미리 insert 돼 있던 다큐먼트
        orderedBulkOps.insert(MyTxTestDocument(name = "3rd", desc = "3rd desc"))  // insert 안됨
        orderedBulkOps.insert(MyTxTestDocument(name = "4th", desc = "4th desc"))  // insert 안됨

        verify(orderedBulkOps, expected = 1)
    }

    @Transactional(transactionManager = "mongoTransactionManager")
    fun unOrderedBatchInsert() {
        setup()

        log.info("Transactional unOrderedBatchInsert 시작")
        val unOrderedBulkOps = mongoTemplate.bulkOps(BulkOperations.BulkMode.UNORDERED, MyTxTestDocument::class.java)
        (1..15).forEach { num ->
            unOrderedBulkOps.insert(MyTxTestDocument(name = "${num}th", desc = "${num}th desc"))  // 롤백 됨
        }
        unOrderedBulkOps.insert(dupDocument())  // 미리 insert 돼 있던 다큐먼트
        (16..30).forEach { num ->
            unOrderedBulkOps.insert(MyTxTestDocument(name = "${num}th", desc = "${num}th desc"))  // 롤백 됨
        }

        verify(unOrderedBulkOps, expected = 1)
    }

    private fun printBulkWriteResult(bulkWriteResult: BulkWriteResult) {
        log.info("bulkWriteResult.insertedCount: ${bulkWriteResult.insertedCount}")
        log.info("bulkWriteResult.deletedCount: ${bulkWriteResult.deletedCount}")
        log.info("bulkWriteResult.matchedCount: ${bulkWriteResult.matchedCount}")
        log.info("bulkWriteResult.modifiedCount: ${bulkWriteResult.modifiedCount}")
    }

    private fun setup() {
        mongoTemplate.remove(MyTxTestDocument::class.java)
        mongoTemplate.insert(dupDocument())
    }

    private fun dupDocument(): MyTxTestDocument = MyTxTestDocument(name = "abc", desc = "def")

    private fun verify(bulkOperations: BulkOperations, expected: Int) {
        try {
            bulkOperations.execute()
        } catch (t: Throwable) {
            t.printStackTrace()
        } finally {
            val all = mongoTemplate.findAll(MyTxTestDocument::class.java)
            check(all.size == expected) {
                log.error("예상된 다큐먼트 수 $expected != 실제 다큐먼트 수 ${all.size}")
            }
        }
    }

    private val log = LoggerFactory.getLogger(javaClass)
}

```
