# QueryDSL Join Alias

동일한 테이블을 사용하는 1차 카테고리와 그 하위인 2차 카테고리 엔티티가 있다.

```kotlin
@Entity
@Table(
    indexes = [Index(name = "IDX_CATEGORY_PARENT_ID", columnList = "parentId", unique = false)]
)
class Category(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    val name: String,
    val parentId: Long? = null,
) {
}
```

카테고리 Id를 포함하는 상품 엔티티가 있다.

```kotlin
@Entity
@Table
class Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    val name: String,
    val category1Id: Long? = null,
    val category2Id: Long? = null,
) {
}

```

아래와 같이 이름을 포함해서 상품 정보를 반환하려면 두 테이블을 조인해야 한다. 카테고리가 2개 있기 때문에 카테고리 테이블과 2번 조인해야 하는데 카테고리 테이블은 1개다.

```kotlin
data class ProductOut @QueryProjection constructor(    
    val id: Long? = null,
    val name: String,
    val category1Id: Long? = null,
    val category1Name: String? = null,
    val category2Id: Long? = null,
    val category2Name: String? = null,
) {
}

```

QueryDsl 을 써서 아래와 같이 하려는데 1개의 카테고리 테이블을 2회 조인하려니 뭔가 어색하다.

```kotlin
class ProductRepositoryImpl: QuerydslRepositorySupport(Product::class.java), ProductRepositoryCustom {

    private val qProduct = QProduct.product
    private val qCategory = QCategory.category
    

    override fun findPagedProducts(pageable: Pageable): Page<ProductOut> {
        val result = from(qProduct)
            .innerJoin(qCategory).on(qProduct.category1Id.eq(qCategory.id))  // 어떻게 구분하지
            .innerJoin(qCategory).on(qProduct.category2Id.eq(qCategory.id))  // 어떻게 구분하지
            .select(
                QProductPrj(
                    qProduct.id,
                    qProduct.name,
                    qProductCategory.id,    // 어떻게 구분하지
                    qProductCategory.name,  // 어떻게 구분하지
                    qProductCategory.id,    // 어떻게 구분하지
                    qProductCategory.name,  // 어떻게 구분하지
                )
            )
            .fetchAll()
        val pagedResult = querydsl!!.applyPagination(pageable, result).fetch()
        return PageableExecutionUtils.getPage(pagedResult, pageable) { result.fetchCount() }
    }
}


```

이럴 때 `QCategory` 생성자를 사용해서 Alias 를 만들어서 `QCategory.category` 대신 사용하면 쉽게 해결할 수 있다.

```kotlin
class ProductRepositoryImpl: QuerydslRepositorySupport(Product::class.java), ProductRepositoryCustom {

    private val qProduct = QProduct.product
    // private val qCategory = QCategory.category
    private val qCategory1 = QCategory("category111")  // Alias
    private val qCategory2 = QCategory("category222")  // Alias
    

    override fun findPagedProducts(pageable: Pageable): Page<ProductOut> {
        val result = from(qProduct)
            .innerJoin(qCategory1).on(qProduct.category1Id.eq(qCategory1.id))
            .innerJoin(qCategory2).on(qProduct.category2Id.eq(qCategory2.id))
            .select(
                QProductPrj(
                    qProduct.id,
                    qProduct.name,
                    qProductCategory1.id,
                    qProductCategory1.name,
                    qProductCategory2.id,
                    qProductCategory2.name,
                )
            )
            .fetchAll()
        val pagedResult = querydsl!!.applyPagination(pageable, result).fetch()
        return PageableExecutionUtils.getPage(pagedResult, pageable) { result.fetchCount() }
    }
}


```

