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

## suspendCoroutine

```kotlin
public suspend inline fun <T> suspendCoroutine(
    crossinline block: (Continuation<T>) -> Unit
): T {
    // 함수가 어떻게 호출되는지에 대한 정보를 컴파일러에 제공합니다.
    // `callsInPlace(block, InvocationKind.EXACTLY_ONCE)`는 block 람다가 정확히 한 번 호출되어야 함을 나타냅니다.
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    
    // `suspendCoroutineUninterceptedOrReturn`은 일시 정지 지점을 만듭니다.
    return suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
        // `c.intercepted()`를 통해 현재 `continuation`을 가져오고 그것을 `SafeContinuation`로 안전하게 래핑합니다.
        val safe = SafeContinuation(c.intercepted())
        // `block`에 정의된 코드 실행하며 현재 `continuation`을 전달해줍니다.
        block(safe)
        // `SafeContinuation`에서 실행된 코드의 결과를 가져오거나 예외를 던집니다.
        safe.getOrThrow()
    }
}
```

---------------------------------------------------------------------------------

## suspendCoroutineUninterceptedOrReturn

```kotlin
public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(
    crossinline block: (Continuation<T>) -> Any?
): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    throw NotImplementedError("Implementation of suspendCoroutineUninterceptedOrReturn is intrinsic")
}
```

suspend 함수 내 현재 `continuation` 인스턴스를 얻고 코루틴을 일시 정지, 또는 정지 없이 즉시 결과를 반환합니다.

이때 코루틴의 현재 상태가 [COROUTINE-SUSPENDED](#CoroutineSingletons) 반환되면 실행이 정지 되어 즉시 결과를 반환하지 않았음을 의미합니다.
이때 `Continuation`은 결과를 사용할 수 있으면 코루틴을 재개하여야 합니다.

반면, `block` 반환 값은 해당 suspend 함수의 결과를 나타내며 실행이 중단되지 않았음을 의미합니다.
또한 `block` 결과 타입이 `Any?`로 선언되어있어 개발자가 어떻게 작성하였느냐에 따라 달라질 수 있습니다.

`Continuation`을 직접 재개할 때 호출 스레드에서 직접 수행되며, 호출한 컨텍스트가 적절하게 확립되었는지 확인해야 합니다.
중단된 `continuation`을 얻고 싶으면 `Continuation.intercepted`을 사용하여 얻을 수 있습니다.

또한 suspend 함수가 실행되는 동일한 스택 프레임에서 [동기적으로 특정 함수를 호출하는 것은 권장되지 않으며](#동기적-호출-문제점),
현재 `continaution` 인스턴스를 안전하게 얻기 위해 [suspendCoroutine](#suspendcoroutine)을 사용하는 것이 좋습니다.

### 동기적 호출 문제점

동기적 호출은 현재 실행 중인 함수나 메서드가 다른 함수나 메서드를 호출하고 그 호출이 완료될 때까지 기다리는 방식입니다.
이는 호출된 함수가 완료되어 결과를 반환할 때까지 현재 실행 중인 함수가 일시 정지되어 기다리게 됩니다.

#### 재귀의 위험성

동기적으로 자기 자신을 호출하는 재귀 함수는 `StackOverFlow`를 일으킬 위험이 있습니다. 
코루틴의 경우, 동일한 스택 프레임에서 다른 suspend 함수를 동기적으로 호출하면 예상치 못한 재귀 상황이 발생할 수 있으며, 이로 인해 `StackOverFlow`가 발생할 수 있습니다.

#### 비동기 작업과의 충돌

코루틴은 비동기 작업을 용이하게 만드는 것이 목적입니다.
동기적 호출은 현재 코루틴을 차단하게 되므로 이는 코루틴의 본래 목적과 상충됩니다.
이로 인해 비효율적인 실행이 발생하거나, 복잡한 상태 관리가 필요할 수 있습니다.

#### 컨텍스트 관리

suspend 함수는 특정 컨텍스트에서 실행됩니다.
동기적 호출 시 현재 suspend 함수와 동일한 컨텍스트에서 다른 suspend 함수가 실행되므로, 예상치 못한 부작이나 컨텍스트의 충돌이 발생될 수 있습니다.