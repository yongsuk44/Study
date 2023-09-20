# Select

코루틴의 `select` 함수는 여러 코루틴 중 어느 것이 먼저 완료되는지 기다리는 기능을 제공합니다.  
이를 통해 여러 작업 중 빠르게 완료되는 것만을 대기하고 결과를 처리할 수 있습니다.  
추가적으로, 여러 채널 중에서 데이터를 전송하거나 수신할 수 있는 첫 번째 채널을 선택하는 것도 가능합니다.

`select` 기능은 다음과 같은 상황에서 유용합니다.
- 여러 작업을 동시에 처리하면서 그 중 하나의 결과만을 기다릴 때
- 여러 채널 중 특정 조건을 만족하는 채널에서만 데이터를 처리하고 싶을 때

예를 들면, 다수의 Rest API 중 가장 빠르게 응답이 오는 서비스만을 선택하여 처리하거나, 
여러 데이터 소스 중 데이터가 준비된 소스만을 선택하여 데이터를 수집하는 등의 작업에 활용할 수 있습니다.

------------------------------------------------------------------------------

## Selecting deferred values

`Deferred`는 코루틴에서 비동기 작업의 결과를 대표하는 타입으로 `async`에서 작업을 시작하고 그 결과를 `Deferred`로 반환합니다.  
그러나 때떄로 여러 비동기 작업 중 가장 먼저 완료되는 것의 결과만을 원할 수 있습니다.

이러한 상황에서 `select`를 사용하면 여러 `Deferred` 값 중 가장 먼저 완료되는 것을 선택하고 그 결과를 반환할 수 있습니다.  
`onAwait`를 사용하여 `Deferred` 값이 완료될 때의 처리를 람다로 정의할 수 있으며, `select`는 이러한 `onAwait` 블록 중 가장 먼저 완료되는 것을 선택합니다.

```kotlin
suspend fun requestData1(): String {
    delay(100_000)
    return "Data1"
}

suspend fun requestData2(): String {
    delay(1_000)
    return "Data2"
}

val scope = CoroutineScope(SupervisorJob())

suspend fun askMultipleForData(): String {
    val defData1 = scope.async { requestData1()}
    val defData2 = scope.async { requestData2()}
    
    return select<String> {
        defData1.onAwait { it }
        defData2.onAwait { it }
    }
}

suspend fun main() = coruotineScope {
    println(askMultipleForData())
}

// 1s delay
// Data2
```

위 예제 코드를 보면 `async`는 외부 스코프에서 시작하였기에 `main()`의 코루틴 스코프에 연결되어 있지 않아 코루틴 취소에 영향을 받지 않습니다.

이럴 경우 리소스의 낭비나 예기치 못한 동작을 초래할 수 있습니다.  
예를 들어 여러 데이터 소스에서 데이터를 가져오기 위해 `async`에서 작업을 시작했는데, 
위 예시처럼 가장 빨리 반환되는 결과만 필요한 경우 나머지 작업들이 계속 실행되는 것은 비효율적입니다.

그렇다고 하여 `coroutineScope`에서 `async` 작업할 경우, 더 빨리 완료된 작업의 결과만 필요한 경우에는 부적합 할 수 있습니다.  
왜냐하면 `coroutineScope`에서 시작된 모든 `Job`은 모든 자식 코루틴들이 완료될 때까지 대기하기 때문에 `async` 작업이 완료되지 않으면 `coroutineScope`도 완료되지 않기 때문입니다. 

위와 같은 이유로 위 코드에서 외부 스코프를 제거하여 실행하게 되면 1s가 아닌 100s를 대기한 후 데이터를 출력할 것입니다. 

```kotlin
suspend fun askMultipleForData(): String {
    val defData1 = scope.async { requestData1()}
    val defData2 = scope.async { requestData2()}
    
    return select<String> {
        defData1.onAwait { it }
        defData2.onAwait { it }
    }
}

suspend fun main() = coruotineScope {
    println(askMultipleForData())
}

// 100s delay
// Data2
```

그러나 위에서 언급했듯이, 외부 스코프에서 `async`를 시작하면 해당 스코프의 생명주기와 연결되어 있지 않기에 취소에 영향을 받지 않게됩니다.  
따라서 `select`로 결과를 선택한 후 나머지 `async` 작업들이 계속 실행되게 됩니다.

이 문제를 해결하기 위해, 결과를 선택한 후 명시적으로 `async` 작업들을 취소해야 합니다.  
따라서 `select`로 결과를 선택한 후 `also`를 사용하여 나머지 `async` 작업을 취소하는 것은 좋은 방법이 될 수 있습니다.

```kotlin
suspend fun askMultipleForData(): String = coroutineScope {
    select<String> {
        async { requestData1() }.onAwait { it }
        async { requestData2() }.onAwait { it }
    }.also { coroutineContext.cancelChildren() }
}

suspend fun main() = coroutineScope {
    println(askMultipleForData())
}

// 1s delay
// Data2
```