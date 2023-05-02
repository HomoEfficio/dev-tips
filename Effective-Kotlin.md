# Effective Koltlin

# Item 01 - 가변성을 제한하라

## val vs immutability

- 읽기 전용(read-only)과 불변성(immutability)는 다르다. val 은 읽기 전용이며 불변성을 보장하지 않는다.

    ```kotlin
    var name = "Efficio"
    val fullName
        get() = "Homo $name"
    fun showFullName() = fullName

    println(fullName)  // Homo Efficio

    name = "Ludens"
    println(fullName)  // Homo Ludens
    println(showFullName())  // Homo Ludens
    ```

- 하지만 아래와 같이 `get()`을 사용하지 않으면 fullName 은 불변이기도 하다.

    ```kotlin
    var name = "Efficio"
    val fullName = "Homo $name". // 여기!!
    fun showFullName() = fullName

    println(fullName)  // Homo Efficio

    name = "Ludens"
    println(fullName)  // Homo Efficio
    println(showFullName())  // Homo Efficio
    ```

- (HomoEfficio) `val`만으로 값을 읽을 수 있고, `get()`을 사용하면 의도하지 않는 가변성이 생길 수 있으므로 특별한 이유가 없는 한 `get()`을 사용하지 않는 게 좋겠다.

## val mutableCollection vs var immutableCollection

- `val mutableCollection`의 변경 지점은 mutableCollection 의 메서드
  - mutableCollection 의 메서드 구현에 의존

    ```kotlin
    val books = mutableListOf<String>()
    books.add("book1")  // add() 메서드 구현에 의해 변경
    ```

- `var immutableCollection`의 변경 지점은 새 immutableCollection 가 할당되는 지점
  - mutableCollection 의 메서드 구현에 의존하지 않음
    - Delegates 사용 가능

    ```kotlin
    var books by Delegates.observable(listOf<String>()) { _, old, new ->
        println("Books changed, old: $old, new: $new")
    }

    books += "book1"  // Books changed, old: [], new: [book1]
    books += "book2"  // Books changed, old: [book1], new: [book1, book2]
    ```

- (HomoEfficio) `var immutableCollection` 방식은 새 immutableCollection 생성/할당이 수행되므로 변경이 굉장히 빈번한 상황에서는 성능 영향이 있을 수 있다.



# Item 05 - 예외를 활용해 코드에 제한을 걸어라

- `require(predicate)` 은 IllegalArgumentException 발생
- `check(predicate)`은 IllegalStateException 발생
- `assert(predicate)`은 `-ea` 옵션을 줄 때만 동작하며 RuntimeException 발생
- `nullable ?: run { log.debug("어쩌구") }` 는 `if (nullable == null) { log.debug("어쩌구") }` 보다 간결
- (HomoEfficio) 좋기는 한데 던져지는 예외가 고정돼 있어서 아쉽다.



# Item 06 - 사용자 정의 오류보다는 표준 오류를 사용하라

- IllegalArgumentException, IllegalStateException 등 개발자들에게 대체로 잘 알려진 표준 오류를 사용하는 것이 좋다.
- (HomoEfficio) 표준 오류는 다양한 문맥 정보를 담기에는 부족
  - 필요한 게 '잘 알려진'이라면 반드시 표준 오류일 필요는 없고 이름을 잘 지은 커스텀 오류로도 충분



# Item 09 - use를 사용하여 리소스를 닫아라

- Closeable을 상속받는 리소스에 대해 `resource.use { doSomething() }` 사용하면 리소스 자동 반납
    - 자바의 `try (resource) { doSomething() }` 과 같은 효과
- (HomoEfficio) `use`를 쓰면 안 되는 상황도 있으며, `useLines`는 대용량 파일에 대해서는 성능 저하 위험: https://homoefficio.github.io/2016/08/13/대용량-파일을-AsynchronousFileChannel로-다뤄보기/





