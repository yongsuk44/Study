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