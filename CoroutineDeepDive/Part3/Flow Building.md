# Flow building

## Flow from raw values

`Flow`를 생성하는 가장 간단한 방법은 `flowOf` 함수를 사용하는 것입니다.  
`flowOf`는 주어진 원시 값들로 `Flow`를 생성하며 이는 리스트를 생성하는 `listOf`와 유사한 원리로 동작합니다.

```kotlin
flowOf(1, 2, 3, 4, 5).collect { print(it) }
// 12345
```

값이 없는 Flow를 생성하려면 `emptyFlow`를 사용할 수 있습니다.

```kotlin
emptyFlow<Int>().collect { print(it) }
// nothing
```

------------------------------------------------------------------

## Converters

`asFlow`는 `Iterable`, `Iterator`, `Sequence`와 같은 컬렉션 형태의 데이터 구조를 `Flow`로 변환하는 데 사용됩니다.

```kotlin
listOf(1, 2, 3, 4, 5)
    // setOf(1, 2, 3, 4, 5)
    // sequenceOf(1, 2, 3, 4, 5)
    .asFlow()
    .collect { print(it) }
// 12345
```

------------------------------------------------------------------

## Converting a function to a flow

`Flow`는 때때로 기존의 suspending 함수나 일반 함수의 결과를 `Flow`로 변환해야 하는 경우가 있습니다.  
이렇게 변환된 `Flow`는 해당 함수의 결과값만을 포함하게 됩니다.

`asFlow`를 통해 이러한 변환이 가능하며, 이 함수는 `suspend () -> T` 타입의 suspend 함수나 `() -> T` 타입의 일반 함수를 받아 `Flow<T>`를 반환합니다.

```kotlin
suspend fun main() {
    val function = suspend {
        delay(1000)
        "name"
    }

    function
        .asFlow()
        .collect { print(it) }
}
// 1s delay 후 "name" 출력
```

참조 연산자(`::`)를 사용하여 특정 함수나 속성을 참조할 수 있습니다.  
예를 들어 `fun exampleFunction()`를 참조하려면 `::exampleFunction`을 사용하면 됩니다.  
이렇게 참조된 함수는 변수에 할당되거나, 다른 함수의 인자로 전달될 수 있습니다.

일반 함수를 `asFlow`로 `Flow`로 변환하려면 먼저 이 참조 연산자를 사용하여 해당 함수를 참조 후 `asFlow`를 호출하면 됩니다.

```kotlin
suspend fun getUserName(): String {
    delay(1000)
    return "name"
}

suspend fun main() {
    ::getUserName
        .asFlow()
        .collect { print(it) }
}
// 1s delay 후 "name" 출력
```

------------------------------------------------------------------

## Flow builders

`flow` 빌더는 `Flow`를 생성하는데 가장 일반적인 방법입니다.   
`flow` 빌더는 순차적 데이터 구조나 채널을 생성하는 다른 빌더와 유사한 방식으로 동작하며, 이를 통해 원하는 데이터 스트림을 `Flow` 타입으로 만들 수 있습니다.

`flow` 빌더 내에서 `emit`을 통해 원하는 값을 `Flow`에 추가할 수 있습니다.   
예를 들어 특정 연산 결과나 API 호출 응답 값을 `Flow`에 추가할 수 있습니다.

또한 `emitAll`은 주어진 `Channel`이나 `Flow`의 모든 값을 현재의 `Flow`에 순차적으로 추가합니다.  
`emitAll(flow) == flow.collect { emit(it) }`으로 각각의 값을 순차적으로 가져오는 것과 동일합니다.

```kotlin
fun makeFlow(): Flow<Int> = flow {
    repeat(3) { num ->
        delay(1000)
        emit(num)
    }
}

suspend fun main() {
    makeFlow().collect { print(it) }
}
// 1s delay 후 0
// 1s delay 후 1
// 1s delay 후 2
```

네트워크 API 결과를 페이지별로 요청해야하는 사용자 데이터 스트림을 생성하는 예시입니다.

```kotlin
fun allUsersFlow(
    api: UserApi
): Flow<User> = flow {
    var page = 0
    do {
        val users = api.takePage(page++)
        emitAll(users)
    } while (!users.isNullOrEmpty())
}
```

------------------------------------------------------------------

## Understanding flow builder

`Flow` 생성 함수나 확장 함수들은 대부분 내부적으로 `flow` 빌더를 기반으로 동작합니다.

```kotlin
public fun <T> flowOf(vararg elements: T): Flow<T> = flow {
    for (element in elements) {
        emit(element)
    }
}
```

`flow` 빌더는 내부적으로 매우 간단합니다.  
`flow` 빌더는 `Flow` 인터페이스를 구현하는 객체를 생성하며, 이는 `collect` 메서드 내부에서 `block`을 호출하기만 합니다.

```kotlin
interface FlowCollector<in T> {
    suspend fun emit(value: T)
}
interface Flow<out T> {
    suspend fun collect(collector: FlowCollector<T>)
}

fun <T> flow(
    block: suspend FlowCollector<T>.() -> Unit
): Flow<T> = object : Flow<T> {
    override suspend fun collect(collector: FlowCollector<T>) {
        collector.block()
    }
}

fun main() = runBlocking {
    flow { // 1
        emit("A")
        emit("B")
        emit("C")
    }.collect { value -> // 2 
        println(value)
    }
}
// A
// B
// C
```

`Flow` 빌더 호출 시 실제로 하는 것은 `Flow` 객체를 생성하기만 합니다.
이 객체는 내부적으로 `Flow` 인터페이스를 구현하게 되며, `collect`를 호출할 때 `flow` 빌더 내 정의한 `block` 함수가 실행됩니다.
이 `block` 함수 내에서는 다양한 `Flow` 연산을 수행할 수 있으며, 특히 `emit`을 통해 `Flow`에 값을 추가할 수 있습니다.

`emit`은 `FlowCollector`에 정의되어 있으며, 이 인터페이스 구현은 `2`에서 정의된 람다식에 의해 제공됩니다.
위 예시에서는 `emit` 호출될 때마다 `println(value)`를 실행합니다.

이러한 방식으로 `Flow`는 매우 직관적이며 간단하며 `Flow`에 대한 다양한 확장 함수나 추가 기능들은 모두 이 기본적인 원리를 기반으로 구축됩니다.

------------------------------------------------------------------

## ChannelFlow

`Flow`는 Cold 스트림으로써 자동으로 데이터를 생성하거나 전송하지 않으며, consumer가 데이터를 요청할 때만 값을 생성하고 전달합니다.

예를 들어 페이지별 사용자 데이터를 제공하는 API를 `Flow`로 표현할 때,  
첫 번째 페이지의 데이터만 필요한 상황에서 `Flow`를 사용하면 첫 번째 페이지의 데이터만 요청하고 응답을 받게 됩니다.  
그 후 두 번째 페이지의 데이터가 필요할 때까지 데이터 요청은 발생하지 않습니다.

이러한 특성은 필요한 데이터만 요청하므로 불필요한 네트워크 요청이나 연산을 줄일 수 있어 리소스의 효율적인 사용을 가능하게 합니다.

```kotlin
data class User(val name: String)

interface UserApi {
    suspend fun takePage(pageNumber: Int): List<User>
}

class FakeUserApi : UserApi {
    private val users = List(20) { User("User$it") }
    private val pageSize: Int = 3

    override suspend fun takePage(pageNumber: Int): List<User> {
        delay(1000)
        return users
            .drop(pageSize * pageNumber)
            .take(pageSize)
    }
}

fun allUsersFlow(api: UserApi): Flow<User> = flow {
    var page = 0
    do {
        println("Fetching page $page")
        val users = api.takePage(page++)
        emitAll(users.asFlow())
    } while (!users.isNullOrEmpty())
}

suspend fun main() {
    val api = FakeUserApi()
    val users = allUsersFlow(api)
    val user = users
        .first {
            println("Checking $it")
            delay(1000)
            it.name == "User3"
        }

    println(user)
}
// Fetching page 0
// 1s delay
// Checking User(name=User0)
// 1s delay
// Checking User(name=User1)
// 1s delay
// Checking User(name=User2)
// 1s delay
// Fetching page 1
// 1s delay
// Checking User(name=User3)
// 1s delay
// User(name=User3)
```

`channelFlow`는 `Flow`와 `Channel`의 특징을 결합한 것으로, 2가지 특징을 모두 갖고 있습니다.  
`Flow`는 Cold 스트림으로 요청에 따라 데이터를 생성하는 반면, `Channel`은 Hot 스트림으로 독립적으로 데이터를 생성하고 전달합니다.

`channelFlow`는 `Flow`와 같이 `collect`와 같은 터미널 연산으로 시작되지만, `Channel`처럼 독립적으로 데이터를 생성합니다.  
이렇게 생성된 데이터는 별도의 코루틴에서 처리되므로, 데이터의 요청과 처리가 동시에 이루어질 수 있습니다.

이러한 특징으로 `channelFlow`를 활용하면 데이터를 미리 생성하거나, 다양한 소스에서 데이터를 동시에 처리하는 등의 복잡한 작업을 수행할 수 있습니다.
예를 들어 네트워크 API에서 페이지 별 데이터를 가져오는 작업과 이 데이터를 처리하는 작업을 동시에 수행하고자 할 때 `channelFlow`를 사용할 수 있습니다.

```kotlin
fun allUsersFlow(api: UserApi): Flow<User> = channelFlow {
    var page = 0
    do {
        println("Fetching page $page")
        val users = api.takePage(page++)
        users.forEach { send(it) }
    } while (!users.isNullOrEmpty())
}

suspend fun main() {
    val api = FakeUserApi()
    val users = allUsersFlow(api)
    val user = users
        .first {
            println("Checking $it")
            delay(1000)
            it.name == "User3"
        }
    println(user)
}

// Fetching page 0
// 1s delay
// Checking User(name=User0)
// Fetching page 1
// 1s delay
// Checking User(name=User1)
// Fetching page 2
// 1s delay
// Checking User(name=User2)
// Fetching page 3
// 1s delay
// Checking User(name=User3)
// Fetching page 4
// 1s delay
// User(name=User3)
```

`channelFlow` 사용 시 내부에서 `ProducerScope<T>`에 작업을 수행하며, 이 `ProducerScope`는 코루틴의 생성과 생명주기를 관리할 수 있는 환경을 제공합니다.

`ProducerScope`는 `CoroutineScope`를 구현하기에, 코루틴을 시작하거나 다양한 코루틴 연산을 수행할 수 있습니다.  
`ProducerScope` 내부에서는 데이터를 생성하고 전송하는데 `emit` 대신 `send`를 사용합니다.

또한 `ProducerScope` 내의 `SendChannel`의 다양한 함수를 통해 채널을 닫거나, 특정 조건에 따라 데이터 전송을 일시 정지하는 등 채널의 동작을 직접 제어할 수 있습니다.

```kotlin
interface ProducerScope<in E> : CoroutineScope, SendChannel<E> {
    val channel: SendChannel<E>
}
```

`channelFlow`의 주요 장점 중 하나는 내부적으로 코루틴 스코프를 생성하기 때문에 독립적으로 값을 계산하고 처리할 수 있는 환경을 제공한다는 것입니다.
따라서 `channelFlow` 내부에서 다양한 코루틴 빌더를 직접 사용하여 별도의 코루틴을 시작할 수 있습니다.

반면 `flow` 빌더는 이러한 코루틴 스코프를 자동으로 생성하지 않기 때문에, 내부에서 별도의 코루틴을 직접 시작하는 것이 지원되지 않습니다.
따라서 독립적인 작업과 병렬 처리가 필요한 경우 `channelFlow`를 사용하는 것이 적절합니다.

```kotlin
fun <T> Flow<T>.merge(other: Flow<T>): Flow<T> = channelFlow {
    launch { collect { send(it) } }
    other.collect { send(it) }
}

fun <T> contextualFlow(): Flow<T> = channelFlow {
    launch(Dispatchers.IO) { send(computeIoValue()) }
    launch(Dispatchers.Default) { send(computeCpuValue()) }
}
```

------------------------------------------------------------------

## callbackFlow

`callbackFlow`는 UI 이벤트, 네트워크 응답과 같은 비동기적인 콜백을 `Flow`로 표현하고 처리하고자 할 때 사용할 수 있습니다.

기본적으로 `channelFlow`와 `callbackFlow` 둘다 독립적인 코루틴 환경에서 데이터를 생성하고 전달하는 것을 지원합니다.  
그러나 `callbackFlow`는 콜백 기반의 코드와의 통합을 목적으로 설계되었습니다.

`callbackFlow`에서는 `ProducerScope<T>`를 통해 데이터 스트림을 생성하고 관리합니다.  
여기서 제공되는 다양한 함수들을 통해 콜백 기반 비동기 작업을 `Flow`로 변환하고 관리할 수 있습니다.

- `awaitClose` : 코루틴이 콜백을 등록한 후 즉시 종료되는 것을 방지하기 위해 사용됩니다. 이 함수는 채널이 다른 방법으로 닫힐 때까지 코루틴을 일시 중지 상태로 유지합니다.
- `trySendBlocking` : 일반적인 `send`와 달리, 코루틴의 일시 중지 없이 값을 채널에 전송합니다. 이에 따라 일반 함수 내에서도 사용할 수 있습니다.
- `close` : 채널을 종료하고, 해당 `Flow`의 데이터 전송을 중단합니다.
- `cancel` : 예외와 함께 채널을 종료합니다. 이렇게 하면 `Flow` 수신자에게 예외가 전달되어 오류 처리가 가능합니다.

```kotlin
fun flowFrom(api: CallbackBasedApi): Flow<T> = callbackFlow {
    val callback = object : Callback {
        override fun onNextValue(value: T) {
            trySendBlocking(value)
        }
        override fun onApiError(cause: Throwable) {
            cancel(CancellationExcpetion("Api Error", cause))
        }

        override fun onCompleted() {
            channel.close()
        }
    }
    api.register(callback)
    awaitClose { api.unregister(callback) }
}
```