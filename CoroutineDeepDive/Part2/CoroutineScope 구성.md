# Constructing a coroutine scope

## CoroutineScope factory function

`CoroutineScope`는 단일 프로퍼티 `coroutineContext`를 지닌 인터페이스 입니다.

```kotlin
interface CoroutineScope {
    val coroutineContext: CoroutineContext
}
```

따라서 이 인터페이스를 구현하는 클래스를 정의하고 그 안에서 직접 코루틴 빌더를 호출할 수 있습니다.

```kotlin
class SomeTest : CoroutineScope {
    override val coroutinecontext: Coroutinecontext = Job()

    fun onStart() {
        launch {
            // ...
        }
    }
}
```

그러나 위와 같이 클래스가 직접 `CoroutineScope`를 구현하면 클래스 안에서 다른 `CoroutineScope` 메서드를 직접 호출할 수 있습니다.   
만약 해당 스코프에 `cancel`을 호출하면 전체 스코프에 있는 모든 코루틴들이 취소될 수 있는 문제가 발생될 수 있습니다.

이로 인해 대부분의 경우, 프로퍼티로 코루틴 스코프를 객체로 가지고 있고 이 프로퍼티를 활용하여 코루틴 빌더를 호출하는 방식을 선호합니다.

```kotlin
class SomeTest {
    val scope: CoroutineScope = ...

    fun onStart() {
        scope.launch {

        }
    }
}
```

코루틴 스코프 객체를 생성하는 가장 간편한 방법은 `CoroutineScope` 팩토리 함수를 사용하는 것입니다.

이 함수는 제공된 컨텍스트로 스코프를 생성합니다. (만약 컨텍스트에 `Job`이 없다면 구조화된 동시성을 위해 `Job`을 생성합니다.)

```kotlin
fun CoroutineScope(
    context: CoroutineContext
): CoroutineScope = ContextScope(
    if (context[Job] != null) context
    else context + Job()
)

internal class ContextScope(
    context: CoroutineContext
) : CoroutineScope {
    override val coroutinecontext: CoroutineContext = context
    override fun toString(): String = "CoroutineScope(coroutineContext = $coroutineContext)"
}
```