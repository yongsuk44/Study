# Flow introduction

`Flow`는 여러 값들의 연속적인 스트림을 표현하는데, 이 값들은 비동기적으로 계산됩니다.

`Flow` 인터페이스는 이러한 값을 흘러가는 대로 수집하는 기능만을 제공합니다.  
특히, `collect()`는 컬렉션에서의 `forEach`와 유사한 역할을 하며 흐름 안에 있는 각 요소에 대해 특정 작업을 수행할 때 `collect`를 사용합니다.

```kotlin
interface Flow<out T> {
    suspend fun collect(collector: FlowCollector<T>)
}
```

`Flow`에는 `collect`만 멤버 함수로 존재하며, 다른 함수들은 확장 함수로 정의되어 있습니다.  
이 특징은 `Iterable`과 `Sequence`와 유사하며, 이 둘은 `iterator`만을 멤버 함수로 가지고 있습니다.

```kotlin
interface Iterable<out T> {
    operator fun iterator(): Iterator<T>
}

interface Sequence<out T> {
    operator fun iterator(): Iterator<T>
}
```

---

## The characteristics of Flow

Flow의 터미널 작업(`collect` 등)은 스레드를 차단하는 것이 아닌, 코루틴을 중지합니다.
또한 현재 코루틴 컨텍스트를 존중하여 예외를 처리하는 코루틴의 다른 기능들을 지원합니다.

Flow는 코루틴의 취소 메커니즘을 따르며, 부모 코루틴이 취소된 경우, 그 안에서 실행 중인 Flow 연산도 함께 취소됩니다.

Flow 빌더인 `flow`는 자체적으로 중단할 수 있는 함수가 아니며, 특별한 코루틴 스코프 없이 사용할 수 있습니다.
하지만 터미널 연산은 코루틴을 중지하며 터미널 연산은 실행되는 코루틴과 연결됩니다. 

아래 예제는 `collect`에서 `flow` 비럳 내의 람다로 `CoroutineName` 컨텍스트가 전달되는지를 보여줍니다.
또한 `launch` 취소가 `flow` 취소로 이어지는 것도 보여줍니다.

```kotlin
fun usersFlow(): Flow<String> = flow {
    repeat(3) {
        delay(1000)
        val ctx = currentCoroutineContext()
        val name = ctx[CoroutineName]?.name
        emit("Emitting$it user $name")
    }
}

suspend fun main() {
    val users = usersFlow()
    
    withContext(CoroutineName("Name")) {
        val job = launch { users.collect { println(it) } }
        
        launch {
            delay(2100)
            println("I got enough")
            job.cancel()
        }
    }
}

// 1s dealy
// Emitting0 user Name
// 1s dealy
// Emitting1 user Name
// 0.1s delay
// I got enough
```

---

## Flow nomenclature

다음은 Flow의 주요 구성 요소에 대한 설명입니다.

- 모든 Flow는 시작점이 필요하며, `flow` 빌더, 다른 객체로부터의 변환, 또는 헬퍼 함수로부터 시작할 수 있습니다.
- 터미널 연산은 Flow 처리를 마무리하는 마지막 단계로, 대부분 suspending 함수로 작성되어 있어 실행이 일시 중지될 수 있습니다.
- Flow의 시작과 터미널 연산 사이에는 여러 단계의 중간 연산(Intermediate Operations)이 있으며, 이를 통해 데이터를 변형하거나 필터링하는 등의 작업을 할 수 있습니다.

```kotlin
suspend fun main() {
    flow { emit("Msg 1") } // flow Builder
        .onEach { println(it) } // Intermediate Operation
        .onStart { println("Starting") } // Intermediate Operation
        .onCompletion { println("Done") } // Intermediate Operation
        .catch { println("Error") } // Intermediate Operation
        .collect { println("Collected $it") } // Terminal Operation
}
```

---

### Real-life use cases

`Flow`와 `Channel`은 데이터 스트림을 처리하기 위한 메커니즘이지만, 다음의 차이가 있습니다.  
`Flow`는 비동기 데이터 스트림을 나타내며, `Channel`은 여러 코루틴 간 데이터를 교환하는 구조를 나타냅니다.

이처럼 일반적인 사용 사례에서는 `Flow`가 더 많이 사용 됩니다.
예를 들어 UI를 통한 사용자 입력이나 센서에서의 데이터와 같은 연속적인 이벤트 스트림을 관찰할 때, 각각의 관찰자가 해당 이벤트를 독립적으로 받아야할 때 `Flow`가 유용합니다.
또한 아무도 해당 이벤트 스트림을 관찰하고 있지 않을 때, `Flow`는 자동으로 수신을 중단합니다.

`Flow`의 가장 전형적인 사용 사례는 다음과 같습니다.
- 웹소켓, R소켓, 알림 등의 서버 전송 이벤트를 통해 소통되는 메시지 전송 또는 수신
- 버튼 클릭, 텍스트 입력과 같은 연속적인 데이터 스트림으로 간주될 수 있는 사용자 이벤트 관찰
- 모바일 기기에서의 센서(가속도계, GPS 등)에서 수신되는 정보
- 데이터 베이스의 변화 관찰

```kotlin
@Dao
interface MyDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
}
```

또한 `Flow`는 다양항 동시 처리 상황에서도 유용합니다. 
예를 들어 판매자 목록이 있고 각 판매자의 제안을 가져와야 하는 상황이 있다고 가정하면, `Flow`를 사용하지 않고 처리하는 경우 `async`를 사용하여 수행할 수 있습니다.

```kotlin
suspend fun getOffers(
    sellers: List<Seller>
): List<Offer> = coroutineScope {
    sellers
        .map { seller -> async { api.requestOffers(seller.id) } }
        .flatMap { it.await() }
}
```

위 방법의 경우에는 `sellers`의 크기가 크다면 한번에 너무 많은 요청을 보낼 수 있어 서버에 과부하를 줄 수 있습니다.  
물론 저장소에서 제한을 둘 수 있지만, 클라이언트 측에서도 이를 제어하고 싶을 수 있습니다.

이럴 때 `Flow`를 사용하여 동시에 발생하는 호출의 수를 제한하여 사용할 수 있습니다.

```kotlin
suspend fun getOffers(
    sellers: List<Seller>
): List<Offer> = sellers
    .asFlow()
    .flatMapMerge(concurrency = 20) { seller -> 
        suspend { api.requestOffers(seller.id) }.asFlow()
    }
    .toList()
}
```