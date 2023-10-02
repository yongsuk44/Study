# Flow lifecycle functions

`Flow`는 데이터 스트림을 표현하기 위한 도구로, 이 스트림은 파이프와 같은 메커니즘을 가집니다.  
이 파이프 내에서 데이터 요청과 응답은 서로 다른 방향으로 이동합니다.

`Flow` 수집 과정 중 어떤 문제나 예외가 발생하면 해당 정보는 파이프 내에 전파되며, 이를 통해 중간 동작이나 처리 단계를 종료하게 됩니다.  
예를 들어 API 호출을 통해 데이터를 가져오는 `Flow`가 있을 때, 이 API 호출에서 문제가 발생하면 해당 문제의 정보는 `Flow`를 통해 전파되고, 이후의 데이터 처리 단계는 실행되지 않습니다.

더불어 `Flow`는 다양한 이벤트를 감지하고 반응하는 기능을 제공합니다.  
Flow 수집이 시작되는 시점, 데이터가 전달되는 시점, 예외가 발생하는 시점 등 다양한 상황에 대해 로직을 삽입하거나 특정 동작을 수행할 수 있습니다.

---

## onEach

`onEach`는 일시 중지될 수 있고 요소들을 순서대로 처리합니다. 따라서 `onEach`에 `delay` 추가하면 각 값을 지연시킬 수 있습니다.

```kotlin
suspend fun main() {
    flowOf(1, 2, 3, 4)
        .onEach { print(it) }
        .collect() 
    // 1234
    
    flowOf(1,2)
        .onEach { delay(1000) }
        .collect { println(it) }
    // 1s delay 후 1 출력
    // 1s delay 후 2 출력
}
```

---

## onStart

`onStart`는 `Flow` 수집이 시작될 때 동작하는 리스너 함수로, 터미널 연산이 호출될 때 즉시 실행됩니다.  
즉, `Flow`에서 값이 흐르기 시작하기 전, 즉시 실행됩니다.

예를 들어, 특정 API로부터 데이터를 가져오는 `Flow`에서 데이터를 실제로 가져오기 시작하기 전에 로깅이나 초기화 작업을 수행하고자 할 때, `onStart`를 사용할 수 있습니다.

```kotlin
suspend fun main() {
    flowOf(1, 3)
        .onEach { delay(1000) }
        .onStart { println("Starting flow") }
        .collect { println(it) }
}
// Starting flow
// 1s delay 후 1 출력
// 1s delay 후 3 출력

suspend fun main() {
    flowOf(1, 3)
        .onEach { delay(1000) }
        .onStart { emit(0) }
        .collect { println(it) }
}
// 0
// 1s delay 후 1 출력
// 1s delay 후 3 출력
```

---

## onCompletion

`onCompletion`은 `Flow` 수집이 종료 시 실행되는 리스너 함수로 아래 상황과 같이 `Flow`의 종료 시나리오에서 호출됩니다.

- 모든 데이터 전송 후
- 중간에 예외가 발생했을 때
- 코루틴이 취소되었을 때 

예를 들어 `Flow` 수집이 종료되면 로깅을 수행하거나 리소스를 해제할 때 `onCompletion`을 사용할 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    flowOf(1,3)
        .onEach { delay(1000) }
        .onCompletion { println("Done") }
        .collect { println(it) }
}
// 1s delay 후 1 출력
// 1s delay 후 3 출력
// Done

suspend fun main() = coroutineScope {
    val job = launch {
        flowOf(1,3)
            .onEach { delay(1000) }
            .onCompletion { println("Done") }
            .collect { println(it) }
    }
    
    delay(1100)
    job.cancel()
}
// 1s delay 후 1 
// 0.1s delay Done

fun updateNews() {
    scope.launch {
        newsFlow()
            .onStart { showProgress() }
            .onCompletion { hideProgress() }
            .collect { view.showNews(it) }
    }
}
```

---

## onEmpty

`onEmpty`는 `Flow`에서 어떠한 값도 방출되지 않았을 때 실행되는 함수로, `Flow`의 수집 과정 중 어떠한 데이터 전송되지 않고 종료된 경우에만 호출됩니다.

예를 들어 API를 통해 데이터를 가져오는 `Flow`가 있을 떄, API에서 어떠한 결과도 반호나하지 않았을 경우 이를 감지하고 적절한 처리를 하기 위해 `onEmpty`를 사용할 수 있습니다. 
이는 특정 상황에서 데이터가 없는 것이 예외적인 상황이거나 특별한 처리가 필요한 경우에 유용합니다.

```kotlin
suspend fun main() = coroutineScope {
    flow<List<Int>> { delay(1000) }
        .onEmpty { println("Empty") }
        .collect { println(it) }
}
// 1s delay 후 Empty 출력
```

----

## catch

`catch`는 `Flow`에서 예외가 발생했을 때 해당 예외를 처리하기 위한 함수입니다.  
`Flow` 수집 중 어떠한 이유로 예외가 발생하면 해당 예외는 파이프라인을 따라 전파됩니다.  
이 때 `catch`를 사용하면 해당 예외를 감지하고 적절한 처리를 수행할 수 있습니다.

`catch` 내에서 발생한 예외를 인자로 받아들일 수 있으므로, 예외의 종류나 메시지에 따라 다양한 처리를 구현할 수 있습니다.

```kotlin
class MyError: Throwable("My Error")

val flow = flow {
    emit(1)
    emit(2)
    throw MyError()
}

suspend fun main() {
    flow.onEach { println("Got $it") }
        .catch { println("Caught $it") }
        .collect { println("Collected $it") }
}

// Got 1
// Collected 1
// Got 2
// Collected 2
// Caught MyError
```

또한 `catch`는 `Flow`의 동작을 중지하지 않고 계속 진행할 수 있게 합니다.  
예외가 발생한 이후의 `Flow` 단계들은 이미 종료되었을 수 있지만, `catch`내에서 새로운 값을 방출함으로써 `Flow`의 나머지 부분을 계속 실행할 수 있습니다.

```kotlin
val flow = flow {
    emit("msg 1")
    throw MyError()
}

suspend fun main() {
    flow.catch { emit("Error") }
        .collect { println("Collected $it") }
}
// Collected msg 1
// Collected Error
```

`catch`는 `Flow`의 업스트림에 발생한 예외를 처리하는데 사용됩니다.
즉, `catch` 이전에 발생한 예외들만 처리할 수 있으며 이후에 발생된 예외는 `catch`에서 처리되지 않습니다.

```kotlin
suspend fun main() {
    flowOf("Message 1")
        .catch { emit("Error") }
        .onEach { throw Error(it) }
        .collect { println("Collected $it") }
}
// Exception in thread java.lang.Error: Message 1
```

Android에서는 다음과 같이 사용할 수 있습니다.

```kotlin
fun updateNews() {
    scope.launch {
        newsFlow()
            .catch { 
                view.handleError(it)
                emit(emptyList())
            }
            .onStart { showProgress() }
            .onCompletion { hideProgress() }
            .collect { view.showNews(it) }
    }
}
```

-----

## Uncaught exceptions

`Flow` 내에서 잡히지 않은 예외가 발생하면 해당 `Flow`는 즉시 중지됩니다.  
이 때 `collect`는 해당 예외를 다시 던져서 호출자에게 알립니다. 

이러한 예외 처리 방식은 `try-catch`를 사용하여 `Flow`나 코루틴에서 발생하는 예외를 잡아내고 적절한 처리를 수행할 수 있습니다.

```kotlin
val flow = flow {
    emit("msg 1")
    throw MyError()
}

suspend fun main() {
    try {
        flow.collect { println("Collected $it") }
    } catch(e: MyError) {
        println("Caught")
    }
}
// Collected msg 1
// Caught
```

`catch`는 `Flow`의 파이프라인 내에서 예외를 처리하기 위한것이지, 터미널 연산에서 발생하는 예외에 대해서는 동작하지 않습니다.  
이는 `catch`가 `Flow` 파이프라인의 마지막에 위치할 수 없기 떄문입니다.

```kotlin
val flow = flow {
    emit("msg 1")
    emit("msg 2")
}

suspend fun main() {
    flow.onStart { println("Started") }
        .catch { println("Caught $it") }
        .collect { throw MyError() }
}
// Started
// Exceptio in thread ... MyError : My Error
```

따라서 `collect` 함수 내에서 직접 수행하는 대신, `onEach`를 사용하여 해당 작업을 수행하면 발생하는 에외를 `catch`를 통해 쉽게 처리할 수 있습니다. 
이러한 접근 방식을 사용하면 `Flow` 파이프라인 내에서 발생하는 모든 예외를 안전하게 처리할 수 있게됩니다.

```kotlin
val flow = flow {
    emit("msg 1")
    emit("msg 2")
}

suspend fun main() {
    flow.onStart { println("Started") }
        .onEach { throw MyError() }
        .catch { println("Caught $it") }
        .collect()
}
// Started
// Caught MyError
```

-----

## flowOn

`Flow`의 연산들이나 빌더들 내에서 사용되는 일시 중단 함수들은 코루틴 컨텍스트와 비슷한 컨텍스트를 가지고 있습니다.  
이 컨텍스트는 일반적으로 `Flow`를 수집하는 `collect`가 호출되는 위치의 컨텍스트를 사용합니다.

따라서 `Flow` 파이프라인 내에서 어느 위치에서든지 일시 중단 함수를 호출하더라도, 그 함수는 `collect`가 호출되는 컨텍스트에서 실행됩니다.

```kotlin
fun usersFlow(): Flow<String> = flow {
    repeat(2) {
        val ctx = currentCoroutineContext()
        val name = ctx[CoroutineName]?.name
        emit("User$it in $name")
    }
}

suspend fun main() {
    val users = usersFlow()
    withContext(CoroutineName("Name1")) {
        users.collect { println(it) }
    }
    withContext(CoroutineName("Name2")) {
        users.collect { println(it) }
    }
}
// User0 in Name1
// User1 in Name1
// User0 in Name2
// User1 in Name2
```

터미널 연산 호출은 업스트림으로부터 요소를 요청함으로써 컨텍스트를 전파합니다.  
이때 `flowOn`을 사용하면 `Flow`의 특정 부분만 다른 컨텍스트에서 실행되게 할 수 있습니다.

또한 `flowOn`은 `Flow`의 업스트림에 있는 함수들에 대해서만 동작함을 주의해야 합니다.

```kotlin
suspend fun present(place: String, message: String) {
    val ctx = currentCoroutineContext()
    val name = ctx[CoroutineName]?.name
    println("[$name] $message on $place")
}

fun messageFlow(): Flow<String> = flow {
    present("flow builder", "Message")
    emit("Message")
}

suspend fun main() {
    withContext(CoroutineName("Name1")) {
        messageFlow()
            .flowOn(CoroutineName("Name3"))
            .onEach { present("onEach", it) }
            .flowOn(CoroutineName("Name2"))
            .collect { present("collect", it) }
    }
}
// [Name3] Message on flow builder
// [Name2] Message on onEach
// [Name1] Message on collect
```

---

## launchIn

`collect`는 `Flow` 데이터를 수집핟고 처리하는 일시 중지 함수입니다.  
때로는 `Flow`의 데이터 처리를 별도의 코루틴에서 비동기로 실행하고 싶은 경우 `luanch`와 같은 코루틴 빌더와 함께 `collect`를 사용하는 것이 일반적입니다.
그러나 이러한 패턴을 더 간결하게 만들기 위해 `launchIn`이라는 함수를 제공합니다.

`launchIn`은 `collect`를 별도의 코루틴에서 자동으로 실행해주며, 해당 코루틴의 스코프를 인수로 받아 실행합니다.

```kotlin
fun <T> Flow<T>.launchIn(
    scope: CoroutineScope
): Job = scope.launch { collect() }
```

```kotlin
suspend fun main(): Unit = coroutineScope {
    flowOf("User1", "User2")
        .onStart { println("Users:") }
        .onEach { println(it) }
        .launchIn(this)
}
// Users:
// User1
// User2
```