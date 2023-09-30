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