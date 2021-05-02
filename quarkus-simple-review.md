# quarkus 간단 후기

[스프링으로 하는 마이크로서비스 구축](http://www.yes24.com/Product/Goods/95593443) 책의 거의 끝에 나오는 쿠버네티스 관련 내용을 이제야 훑어봤다.  
마이크로서비스를 쿠버네티스 환경으로 이관하면서 다음과 같은 스프링 클라우드 컴포넌트를 쿠버네티스 컴포넌트로 대체하는 내용이 나온다.

- Eureka
- Config
- Gateway

이 3가지는 스프링 클라우드를 사용해서 애플리케이션 수준에서 구현해왔지만 사실 비즈니스 문제보다는 마이크로서비스 구조 자체에서 발생하는 문제를 해결하는 성격의 컴포넌트라서, 애플리케이션 수준이 아니라 인프라 수준에서 해결할 수 있으면 더 좋을 것 같았다.

다행히 쿠버네티스 환경에서는 이런 문제를 쿠버네티스 수준에서 해결할 수 있다. 그래서 위 3가지를 쿠버네티스 컴포넌트로 대체할 수 있으며, 덕분에 스프링 클라우드에 대한 의존성을 조금 내려 놓고 정말로 문제 해결에 맞는 기술 스택을 선택해서 폴리글랏 마이크로서비스를 만들 수 있겠다는 생각이 들었다.

물론 스프링 클라우드를 사용해도 폴리글랏은 가능하다. 해보진 않았지만 스프링 클라우드 기반이 아닌 마이크로서비스에는 Sidecar라는 부가적인 컴포넌트를 또 추가해야 협력이 가능한 걸로 알고 있다.

암튼 이젠 **쿠버네티스 인프라 덕분에 꼭 스프링 부트 기반이 아닌 다른 기술 스택을 사용해도 매끄럽게 폴리글랏 마이크로서비스를 구현할 수 있다.**

그래서 먼저 살펴본 것이 [쿼커스(quarkus)](https://quarkus.io/)다.

쿼커스는 아래와 같이 소개돼 있다.

>**Supersonic Subatomic Java**
>
>A Kubernetes Native Java stack tailored for OpenJDK HotSpot and GraalVM, crafted from the best of breed Java libraries and standards.

한 마디로 **졸라 빠르고 졸라 작은 자바, 킹왕짱 라이브러리와 표준을 골라서 OpenJDK HotSpot, GraalVM에서 씽씽 돌아가는 k8s 깔맞춤 자바 스택**이란 소리다.

조금 더 쉽게 얘기하자면 **스프링이 아니라 [이클립스 버텍스(vert.x)](https://vertx.io/)를 중심으로 하는 자바 스택**이라고 할 수 있겠다.

일반적인 쿼커스 특징이나 소개는 다른 자료에도 많으니 여기에서는 코틀린, 리액티브, 몽고디비를 사용해서 CRUD도 아니고 CR만 하는 초간단 미니 프로젝트 깔짝대면서 느낀 점을 끄적여본다.

Aㅏ 물론 아주 살짝 깔짝대보고 쓰는 거라 잘못된 내용이 포함돼 있을 수도 있다. 언제든 발각되면 고치기로 하고 일단 가즈아ㅏㅏㅏ


## 지리는 Live Coding

수퍼소닉, 서브아토믹, 클라우드 네이티브, k8s 이런 거창하면서도 비구체적인 수식어에는 사실 큰 감흥이 없었다.  
개발자로서 정말 지리는 포인트는 [Live Coding](https://quarkus.io/vision/developer-joy#live-coding)이었다.

애플리케이션을 로컬에서 dev 모드로 실행하면, 그 이후에 메서드 추가/변경뿐 아니라 클래스를 추가/변경하거나 라이브러리를 임포트를 하거나 설정 파일을 수정하거나 등등 변경 작업 후에 수동으로 애플리케이션을 재시작하지 않아도, 애플리케이션에 새 요청이 들어오면 변경 부분만 샥 반영돼서 자동으로 **빠르게** 재시작 된다. 그런데 이 재시작이 대략 2초 이내로 굉장히 빠르다. [JRebel](https://www.jrebel.com/)을 쓰는 게 이런 느낌이었을까?


## Reactive

쿼커스는 버텍스 기반이라고 했다. 버텍스는 쉽게 말해 자바판 node.js라고도 알려졌었는데, 스프링보다도 훨씬 이른 시기부터 비동기, 논블로킹 라이브러리임을 표방해왔다. 그래서 리액티브 지원 생태계도 잘 갖춰져 있다.

### Mutiny

스프링에서 사용되는 리액터 대신에 [뮤티니(Mutiny)](https://smallrye.io/smallrye-mutiny)를 사용한다. 리액터의 Flux, Mono 처럼 뮤티니에도 Multi, Uni 가 있는데, 치이점은 리액터의 Flux, Mono 는 둘다 리액티브 스트림의 Publisher 구현체인데 반해, 뮤티니의 Uni는 Publisher 구현체가 아니다. Uni 같은 경우 스트리밍이 아니라 단건단건 똑 떨어지는 결과에만 관심이 있으므로 굳이 복잡한 리액티브 스트림 구조를 따르지 않고 더 간단하게 처리할 수 있게 하기 위해 Publisher를 구현하지 않았다고 한다.

어쨌든 사용하는 입장에서는 리액티브 어쩌구 api는 일반 문장처럼 느껴지는 평문형 api를 제공한다는 미명하에 비즈니스 해결과 무관한 코드량이 오지게 많아지고 그로 인한 학습 곡선이 높다는 단점이 있는데 뮤티니도 이런 한계에서 크게 벗어나지는 못한 것 같다. 걍 리액터랑 비슷한 다른 놈이고 필요하다면 리액터 타입과 상호 운영도 가능하다. 코드 예제는 아래 몽고 클라이언트에서 볼 수 있다.

### Mongo Client

쿼커스에서 제공하는 리액티브 몽고 클라이언트는 기본적으로는 bson Document와 뮤티니 기반이다. 스프링 데이터 리포지토리 같은 개념은 없고 스프링 데이터 JDBC와 비슷한 API를 제공하며 대략 다음과 같이 생겼다. transform의 람다 인자 등 몇 가지가 좀 번잡스럽게 생겼고 잘못 작성한 걸 수도 있는데, 이건 나도 아직 익숙하지 못해서 그런 것이니, 그보다는 `insertOne`, `onItem`, `transform`, `collect`, `asList` 같은 뮤티니 API를 보면서 감을 잡아 보는 게 목적이다.

```kotlin
// service
fun add(other: Other): Uni<String> {
    val doc = Document()
        .append("message", other.message)
    return getCollection()
        .insertOne(doc)
        .onItem()
        .transform { it.insertedId!!.asObjectId().value.toHexString() }
}

// resource(스프링의 controller에 해당)
@POST
@Consumes(MediaType.APPLICATION_JSON)
fun new(other: Other): Uni<Response> {
    val id: Uni<String> = svc.add(other)
    return id.map { Response.created(URI.create("/others/$it")).build() }
}


// service
fun findAll(): Uni<MutableList<Other>> {
    return getCollection().find()
        .onItem()
        .transform { Other(it.getObjectId("_id").toHexString(), it.getString("message")) }
        .collect().asList()
}

// resource
@GET
fun findAll(): Uni<MutableList<Other>> {
    return svc.findAll()
}
```

하지만 [퍼나쉬(Panache)](https://quarkus.io/guides/mongodb-panache)를 사용하면 스프링 데이터와 JPA에서 익숙했던 Entity, Repository 스타일도 리액티브에서도 사용할 수 있다. 다만 도메인 객체가 퍼나쉬에 의존하는 모양새가 되는 것 같다.


### 코루틴 지원

뮤티니는 코루틴도 플러그인 형태로 지원한다. 그런데 아쉽게도 리소스(스프링의 컨트롤러)에서 suspend 함수 사용을 지원하지 않는다.  
하지만 [쿼커스 2.0 부터는 지원 예정이고, 현재 2.0은 알파 단계](https://github.com/quarkusio/quarkus/releases/tag/2.0.0.Alpha1)다. 그 전까지는 아래와 같이 직접 코루틴 스코프 등을 만들어서 코루틴 코드를 사용할 수 있다.

아래 코드는 [쿼커스 깃헙 이슈 댓글](https://github.com/quarkusio/quarkus/issues/10162#issuecomment-822029518)에서 가져왔다.

```kotlin
package io.homo_efficio

import io.smallrye.mutiny.Uni
import io.smallrye.mutiny.coroutines.asUni
import io.vertx.core.Context
import io.vertx.core.Vertx
import kotlinx.coroutines.*
import java.lang.Runnable
import java.util.concurrent.AbstractExecutorService
import java.util.concurrent.TimeUnit
import javax.enterprise.context.ApplicationScoped
import kotlin.coroutines.CoroutineContext
import kotlin.coroutines.EmptyCoroutineContext

@ApplicationScoped
class MyScope : CoroutineScope {

    override val coroutineContext: CoroutineContext = SupervisorJob()

    @ExperimentalCoroutinesApi
    fun <T> asyncUni(
        context: CoroutineContext = EmptyCoroutineContext,
        start: CoroutineStart = CoroutineStart.DEFAULT,
        block: suspend CoroutineScope.() -> T
    ): Uni<T> {
        val vertxContext = checkNotNull(Vertx.currentContext())
        val dispatcher = VertxCoroutineExecutor(vertxContext).asCoroutineDispatcher()
        return async(context + dispatcher, start, block).asUni()
    }
}

class VertxCoroutineExecutor(
    private val vertxContext: Context
) : AbstractExecutorService() {

    override fun execute(command: Runnable) {
        if (Vertx.currentContext() != vertxContext) {
            vertxContext.runOnContext { command.run() }
        } else {
            command.run()
        }
    }

    override fun shutdown(): Unit = throw UnsupportedOperationException()
    override fun shutdownNow(): MutableList<Runnable> = throw UnsupportedOperationException()
    override fun isShutdown(): Boolean = throw UnsupportedOperationException()
    override fun isTerminated(): Boolean = throw UnsupportedOperationException()
    override fun awaitTermination(timeout: Long, unit: TimeUnit): Boolean = throw UnsupportedOperationException()
}

```

위와 같이 커스텀 스코프를 만들면 아래와 같이 서비스에서는 suspend 함수를, 리소스에서는 커스텀 스코프를 사용해서 서비스의 suspend 함수를 호출하는 코루틴 코드를 사용할 수 있다.

```kotlin
// service
suspend fun add(other: Other): String {  // 여기!! suspend 함수 사용
    val doc = Document()
        .append("message", other.message)
    val addedDoc: InsertOneResult = getCollection()
        .insertOne(doc).awaitSuspending()  // 여기!! awaitSuspending()
    return addedDoc.insertedId!!.asObjectId().value.toHexString()
}

// resource
@POST
@Consumes(MediaType.APPLICATION_JSON)
fun new(other: Other): Uni<Response> {
    return scope.asyncUni {  // 여기!! 커스텀 스코프 사용
        val id = svc.add(other)
        Response.created(URI.create("/others/$id")).build()
    }
}
```

## 개발용 웹 UI

아래와 같이 몇 가지 개발에 활용할 수 있는 웹 UI도 제공해준다.

![Imgur](https://i.imgur.com/X3JRQCn.png)


## 마무리

다른 자료에서 이미 강조하고 있는 Supersonic, Subatomic은 분명한 것 같으니 그러려니 한다. 하지만 [부하가 커지면 오히려 자원 사용량이 높아진다는 의견](https://medium.com/swlh/springboot-vs-quarkus-a-real-life-experiment-be70c021634e)도 있다.

GraalVM에 얹으면 진정한 Supersonic, Subatomic이 된다고는 하지만 스프링 부트라고 가만히 있을까? 그래서 이 부분은 상대적으로 격차가 점점 줄지 않을까 싶다.

완전히 바닥부터 새로 만들었다기 보다 버텍스 중심으로 현존하는 좋은 기술을 짬뽕해서 만들어서 그런지 전반적으로 스프링에 버금갈 정도로 생태계가 잘 갖춰져 있는 걸로 보인다. 다양한 공식 가이드도 스프링 못지 않게 좋다. 다만 서드파티 자료는 아무래도 스프링 자료를 따라갈 수는 없을 듯하니 실제 현업에서 발생하는 자질구레한 다양한 문제 해결에는 살짝 차질이 있을 수도 있다. 
하지만 전반적으로 스프링 부트를 주로 쓰다가 쿼커스로 오더라도 엄청난 진입 장벽을 느낄 정도는 아니다.

**지리는 Live Coding 기능을 생각하면 이 정도 진입 장벽은 넘어야지!**라는 생각마저 들 정도.

리액티브로 가면 마찬가지로 장황해지니, 리액티브를 사용해야 한다면 정식으로 코틀린 코루틴을 사용할 수 있게 되는 2.0이 나올 때까지 기다리는 게 좋겠고, 굳이 리액티브를 쓰지 않는다면 Hibernate도 사용 가능하니 당장 갈아타도 괜찮지 않을까 싶다.

검색해보면 레드햇 코리아에서 만든 한글 자료가 있으니 관심 있으면 ㄱㄱ


