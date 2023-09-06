## CoroutineSingletons

```kotlin
internal enum class CoroutineSingletons {
    COROUTINE_SUSPENDED, UNDECIDED, RESUMED
}
```

현재 코루틴의 상태를 나타내며 다음과 같은 상태를 가집니다.

|          상태           |                                      설명                                      |
|:---------------------:|:----------------------------------------------------------------------------:|
| `COROUTINE_SUSPENDED` |                              코루틴이 현재 일시 정지된 상태                               |
|      `UNDECIDED`      | 코루틴의 상태가 아직 결정되지 않았음을 나타냅니다. 즉, 중단될지, 재개될지, 아니면 계속 진행될지가 아직 결정되지 않았음을 의미합니다. |
|       `RESUMED`       |                           현재 코루틴이 재개된 상태임을 나타냅니다.                            |

또한 다음과 같은 특징을 지니고 있습니다.

- **직렬화** : `enum class CoroutineSingletons` 사용 시 `SafeContinuation`을 모든 종류의 직렬화 프레임워크와 함께 직렬화 할 수 있습니다.
- **디버깅 경험 향상** : `enum class CoroutineSingletons` 사용하면 디버깅 중 객체의 `toString()` 값을 명확하게 볼 수 있으며, 어느 패키지에서 해당 객체가 비롯되었는지를 알 수 있습니다.

---------------------------------------------------------------------------------

## SafeContinuation

```kotlin
internal expect class SafeContinuation<in T> : Continuation<T> {
    
    // 코루틴의 현재 상태를 나타내는 Continuation을 래핑하며, 초기 결과 값을 설정할 수 있습니다
    internal constructor(delegate: Continuation<T>, initialResult: Any?)

    // 초기 결과 값 없이 코루틴의 현재 상태를 나타내는 Continuation을 래핑합니다.
    @PublishedApi internal constructor(delegate: Continuation<T>)

    // 코루틴의 결과를 가져오거나, 예외가 발생한 경우 예외를 던집니다. 
    // 이 메서드는 코루틴의 결과를 안전하게 가져오는 데 사용됩니다.
    @PublishedApi internal fun getOrThrow(): Any?

    // 현재 코루틴의 컨텍스트를 나타내며, 코루틴이 실행되는 환경 정보를 포함하여 스레드, 디스패처, 작업 범위 등을 관리합니다.
    override val context: CoroutineContext

    // 코루틴을 계속해서 재개하며, Result<T>를 통해 성공한 결과 또는 실패한 예외 정보를 전달합니다. 
    override fun resumeWith(result: Result<T>): Unit
}

```

---------------------------------------------------------------------------------

## Mechanism of continuation interception

Continuation Interception 메커니즘은 코루틴의 실행 흐름을 관리하는 데 사용되는 기능입니다.

이 메커니즘은 코루틴이 일시 중단되고 재개될 때 어떠한 일이 발생할 지를 결정합니다.

### Component

- Continuation : 코루틴이 일시 중단된 후 다시 시작할 위치와 상태 정보를 가집니다.
- Interceptor : Continuation을 가로채어 특별한 동작을 추가하거나 변경할 수 있습니다.

### Method

- `interceptContinuation` : 코루틴이 일시 중단될 때 호출되며, 일시 중단된 `Continuation`을 수정할 수 있습니다.
- `releaseInterceptedContinuation` : 일시 중단된 코루틴이 끝났을 때 호출되며, 필요한 정리 작업이 있으면 이 메서드에서 수행할 수 있습니다.

### Flow

1. 코루틴 일시 중단 시 `Continuation` 객체 생성
2. `ContinuationInterceptor`의 `interceptContinuation` 메서드가 이 `Continuation` 객체를 받아서 필요한 작업 수행
3. 코루틴이 재개될 준비가 되면 `Intercepted Continuation`이 실행
4. 코루틴의 실행이 종료되면 `releaseInterceptedContinuation` 메서드가 호출되어 정리 작업 수행


이러한 메커니즘을 통해 코루틴의 디스패치, 코루틴 컨텍스트의 관리, 코루틴의 일시 중단과 재개 등이 유연하게 처리될 수 잇습니다.