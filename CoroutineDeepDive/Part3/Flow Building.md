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