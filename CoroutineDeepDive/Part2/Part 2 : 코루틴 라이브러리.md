# Kotlin Coroutines library

코루틴을 제대로 사용하기 위해 필요한 모든 것을 배우며 아래의 항목들을 정리하는 파트 입니다.

## [Part 2.1 : Coroutine Builder](코루틴%20빌더.md)

suspend 함수는 서로 `Continuation`을 전달하며 호출 스택을 쌓아야 합니다.
이 때문에 suspend 함수 내부에서 일반 함수 호출은 문제가 없지만, 일반 함수에서 suspend 함수의 호출은 IDE에서 오류가 발생됩니다.

즉, suspend 함수는 다른 suspend 함수에서 호출되어야 하며, 이들은 코루틴 빌더로 시작되어야 합니다.

### launch Builder

`launch`는 독립적으로 코루틴을 시작하는데 사용되며, 새로운 스레드를 시작하는 것과 유사한 역할을 합니다.  
그러나 `Thead`와 `launch`는 일시 중단된 상황에서 다르게 처리됩니다.

- 스레드 : 스레드 차단 시 계속해서 비용이 발생
- 코루틴 : 코루틴 정지 시 비용이 거의 발생되지 않음

위와 같이 코루틴 사용 시 비용이 거의 발생되지 않기에 더 효율적이게 됩니다.

`launch`는 `CoroutineScope` 인터페이스의 확장함수로, [구조화된 동시성](../Structured%20Concurrency.md)을 형성합니다.

### runBlocking Builder

`runBlocking`은 코루틴이 중단될 때 시작된 스레드를 차단하는 특별한 빌더입니다.   
이러한 특징은 `Thread.sleep(1000L)`과 `delay(1000L)`이 동일하게 동작하게 하므로, 메인 함수나 단위 테스트와 같이 프로그램이나 테스트가 너무 일찍 종료되는 것을 방지하고자 할 때
유용합니다.

최근에는 다음과 같은 변화가 있었습니다.

- 메인 함수의 경우, 자체를 `suspend` 함수로 정의하여 `runBlocking`을 대체할 수 있습니다.
- 단위 테스트의 경우, `runBlocking` 대신 `runTest`를 사용하여 가상 시간에서 코루틴을 작동시킬 수 있습니다.

그럼에도 `runBlocking`을 계속 사용하는 주된 이유는, 이 빌더가 코루틴의 범위(scope)를 생성하며, 이 범위 내에서 코루틴이 완료될 때까지 현재 스레드를 차단하기 때문입니다.

현대 프로그래밍에서는 드물게 사용되지만, 구조화된 동시성을 도입하게 되면 훨씬 더 유용해질 수 있으며, 특정 상황에서 필요한 동작을 제공합니다.

### async Builder

- `async`는 람다 표현식에 의해 반환되며 `Deferred<T>`를 반환합니다.
- `Deferred`는 값이 준비되면 `await`를 통해 값을 반환합니다.
- `async`는 병렬 처리하기에 적합하며 값이 얻기 전에 `await`을 호출하면 값이 준비될 떄까지 중단됩니다.
- `async`는 값을 생성 하는데 중점을 두고 `launch`는 로직만 실행하는 경우에 적합합니다.

### Structured Concurrency

- `launch`는 `runBlocking`의 자식이 될 수 있으며, 부모 코루틴은 자식 코루틴이 끝날 때 까지 대기합니다.
- 부모 코루틴은 자식 코루틴에게 `Scope`를 제공하고, 자식 코루틴은 이 `Scope`내에서 호출 되며 이로 인해 구조화된 동시성이 구축됩니다.
- 자식 코루틴은 부모 코루틴으로부터 `CoroutineContext`를 상속받으며, 필요한 경우 덮어 쓸 수 있습니다.
- 부모 코루틴 취소 시 자식 코루틴도 취소됨, 자식 코루틴 오류 발생 시 부모 코루틴도 파괴됩니다.
- 다른 코루틴 빌더와 달리 `runBlocking`은 root 코루틴으로만 사용됩니다.

### The Big Picture

- 모든 코루틴은 `CoroutineScope`에서 시작되어야 하며, 이는 `runBlocking` 또는 특정 프레임워크에 의해 제공됩니다.
- suspend 함수 내에서 Scope가 없기에 `coroutineScope` 함수를 사용하여 Scope를 적용해야 합니다.

### using coroutineScope

- `coroutineScope`는 일시 정지 함수 내에서 필요한 Scope를 생성하고, 람다 식에서 반환된 값을 반환하는 일시 정지 함수입니다.
- 이는 코드 구조화와 동시성을 관리하는데 유용하며, 코루틴의 효과적인 관리를 도와줍니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    lanunch {
        delay(1000L)
        print("World!")
    }

    print("Hello,")
}
```

### Summary

| Coroutine Builder | Description                                     |
|-------------------|-------------------------------------------------|
| launch            | 독립적인 코루틴을 시작하며, `Job`을 반환합니다.                   |
| async             | 값을 반환할 때까지 중단되는 병렬 처리에 적합하며, `Deferred`를 반환합니다. |
| runBlocking       | 현재 스레드를 차단하고, 코루틴이 완료될 때까지 대기합니다.               |

- 코루틴은 [구조화된 동시성](../Structured%20Concurrency.md)을 위해 작성되며, 부모-자식 관계를 형성합니다.
- 코루틴은 스레드에 비해 일시 중단되었을 때의 비용이 훨씬 저렴합니다.
- 모든 코루틴은 `coroutineScope`에서 시작되어야 하며, 이는 `runBlocking` 또는 특정 프레임워크에 의해 제공됩니다.
- `coroutineScope()`는 일시 정지 함수 내에서 필요한 Scope를 생성하고, 람다 식에서 반환된 값을 반환하는 일시 정지 함수입니다.

---

## [Part 2.2 : Coroutine Context](코루틴%20컨텍스트.md)

### CoroutienContext Interface

- `CoroutineContext`는 요소와 요소의 집합을 나타내는 인터페이스로 `Job`, `CoroutineName`, `CoroutineDispatcher` 등과 같은 `Element` 인스턴스의 집합입니다.
- 위 요소들은 그 자체로 `CoroutineContext`가 될 수 있으며 이러한 구조는 컨텍스트 관리를 유연하게 만들어 줍니다.
- 위 요소들은 유니크 키를 가지며 이를 통해 요소를 식별하고 고유하게 관리할 수 있습니다.

### Finding elements in CoroutineContext

- `CoroutineContext`는 `map`과 유사한 `Key-Value` 구조와 유사하며, 요소는 `Unique Key`로 식별됩니다.
- `CoroutineContext`는 `get`, `[]` 메서드와 `Key`를 통해 특정 요소를 찾을 수 있고, 찾는 요소가 없는 경우 `null`을 반환 합니다.
- Kotlin에서 클래스 이름은 해당 클래스의 `companion object`에 대한 참조로 작동됩니다. (`ctx[CoroutineName] == ctx[CoroutineName.Key]`)
- `kotlinx.coroutines` 라이브러리에서 `companion object`를 요소의 키로 사용하는 것이 일반적입니다.
- 이름이 `Key`인 `companion object`는 특정 클래스(`CoroutineName`) 또는 인터페이스(`Job`)를 가리킬 수 있고,
  동일한 Key를 사용하여 여러 클래스(`job`,`SupervisorJob` 등)을 가리킬 수 있습니다.

### Adding Contexts

- 앞서 말한것처럼 `CoroutineContext`은 2개의 컨텍스트를 합칠 수 있고 이를 통해 새로운 컨텍스트를 생성할 수 있습니다.
- 2개의 서로 다른 키를 합칠 경우 2가지 키 모두 응답되며, 동일한 키를 가진 다른 요소가 추가되면 `map`과 같이 새로운 요소가 이전 요소를 대체 합니다.

### Empty coroutine context

- `EmptyCoroutineContext`는 그 자체로 어떠한 행동도 하지 않으며, 다른 컨텍스트와 결합하면 결합된 다른 컨텍스트는 원래의 컨텍스트와 동일하게 동작됩니다.
- `EmptyCoroutineContext`는 초기 상태로 사용하거나, 필요에 따라 컨텍스트를 동적으로 확장하고자 할 때 유용합니다.

### Subtracting Elements

- `CoroutineContext`는 `map`과 유사한 `minusKey` 메서드를 제공하며, 이는 특정 키를 가진 요소를 제거합니다.
- `minus` 연산자는 `CoroutineContext`에 대해서 연산자의 의미가 명확하지 않기에 오버로드 되지 않았습니다.

### Coroutine context and builders

- 일반적으로 자식 코루틴은 부모 코루틴으로부터 컨텍스트를 상속받으며 이 컨텍스트를 override 하여 사용할 수 있습니다.
- 보통 코루틴 컨텍스트 계산 시 특정 키에 대해서 여러 컨텍스트가 값이 존재하는 경우 '우선순위'(`child > parent > default`)에 따라 어떤 값을 사용할지 결정합니다.
    - `childContext`와 `parentContext`가 동일한 키를 가지고 있으면, `childContext`의 값이 사용됩니다.
    - `defaultContext`에는 기본 설정들이 존재하며 `ContinuationInterceptor` 값이 없으면 `Dispatchers.Default` 적용되며, 디버그 모드
      시 `CoruotineId`를 적용 합니다.
- `Job`은 코루틴 생명주기를 나타내는 특별한 컨텍스트 요소로, 부모-자식 간의 통신에 사용되는 Mutable한 컨텍스트입니다.

### Accessing context in a suspending function

- `CoroutineScope`는 `coroutineContext` 속성을 통해 현재 코루틴 컨텍스트에 접근할 수 있습니다.
- 일반 suspend 함수는 `coroutineContext`에 [직접 접근하는 것이 불가능](suspend%20함수에서%20CoroutineScope%20직접%20접근을%20막은%20이유.md)
  하므로 `continuation`을 통해 `coroutineContext`에 접근해야 합니다.

### Creating our own context

- Custom `CoroutineContext`를 만들기 위해서는 `CoroutineContext.Element`를 구현하는 클래스를 생성하고 `CoroutineContext.Key<*>`
  프로퍼티와 `compainon object Key`를 정의하여 사용하면 됩니다.
- Custom `CoruotienContext`는 특별하게 따로 정의하지 않는 이상 `CoroutineName`과 유사하게
  동작하며 [보통 코루틴 컨텍스트 계산하는 방식](#coroutine-context-and-builders)과 동일하게 적용됩니다.

### Summary

- `CoroutineContext`는 `Job`, `CoroutineName`, `CoroutineDispatcher` 등의 `Element` 요소들의 집합이며 이들은 유니크 키를 가지고 있어 식별과 관리가
  가능합니다.
- `CoroutineContext`는 `Key-Value` 구조로 되어있어 특정 요소를 키로 찾을 수 있습니다.
- 2개의 `CoroutineContext`를 합쳐 새로운 컨텍스트를 만들 수 있으며, 같은 키를 가진 요소가 존재하면 새 요소가 이전 요소를 대체합니다.
- `EmptyCoroutineContext`는 아무런 행동도 하지 않으며 다른 컨텍스트와 병합하여도 원래 컨텍스트의 특성을 유지합니다.
- `CoroutineContext`에서 `minusKey()`를 통해 특정 키를 가진 요소를 제거할 수 있습니다.
- 자식 코루틴은 부모 코루틴의 컨텍스트를 상속받으며, 여러 컨텍스트에 같은 키에 대해서 값이 존재하면 컨텍스트 우선순위에 따라 값이 적용됩니다.

---

## [Part 2.3 : Jobs and awaiting children](Job과%20자식%20코루틴의%20대기.md)

### What is Job?

- `Job`은 코루틴을 취소하거나, 상태를 추적하는 등의 기능을 제공하는 코루틴 컨텍스트 요소입니다.
- `Job`은 `NEW`, `ACTIVE`, `COMPLETING`, `CANCELLING`, `CANCELLED`, `COMPLETED` 등을 통해 상태를 나타낼 수 있습니다.
- `isActive`, `isCompleted`, `isCancelled` 등의 속성을 통해 `Job`의 상태를 확인할 수 있습니다.

### Coroutine builders create their jobs based on their parent job

- 코루틴 빌더는 자체적으로 `Job`을 생성하며 `Job`은 코루틴 컨텍스트이므로 `coroutineContext[Job]` 또는 확장 프로퍼티인 `job`을 통해 접근할 수 있습니다.
- `Job`은 부모 코루틴에서 자식 코루틴으로 자동으로 상속되지 않은 유일한 코루틴 컨텍스트입니다.
    - 대신 부모-자식 형성을 위해서 자식 코루틴의 `Job`은 부모 코루틴의 `Job`을 참조하여 관계를 형성 합니다.
- 자식 코루틴에서 부모 코루틴의 `Job` 컨텍스트가 새로운 `Job`으로 대체될 경우 구조적 동시성 메커니즘이 형성되지 않기에 여러 문제가 발생될 수 있습니다.
    - 부모 코루틴은 자식 코루틴의 본문이 끝날 때 까지 기다리지 않고 종료될 수 있음
    - 자식 코루틴에서 취소 및 예외 처리를 상위 코루틴으로 전달할 수 없음

### Waiting for children

- `Job`은 `Job.join()` 메서드를 통해 호출한 코루틴을 일시 중단시키고 지정된 `Job`이 완료(`Cancelled` or `Completed`)될 때 까지 대기할 수 있습니다.
- `Job`은 `children` 프로퍼티를 통해 자식 코루틴을 참조하여 모든 자식 코루틴이 완료를 확인할 수 있습니다.

### Job factory function

- `Job()` 사용 시 코루틴과 연결되지 않은 `Job` 인스턴스 생성이 가능합니다.
- `Job()`을 통해 생성된 `Job`은 다른 코루틴의 부모로 사용될 수 있지만, 모든 자식 코루틴에 명시적으로 `join`을 호출하지 않으면 `Active` 상태로 유지되는 문제가 발생할 수 있습니다.
- `Job()`은 `CompletableJob`의 하위 인터페이스 타입의 객체를 반환하며, 이를 통해 `complete()`, `completeExceptionally()` 메서드를 사용할 수 있습니다.

| method                                              | description                                                                               |
|-----------------------------------------------------|-------------------------------------------------------------------------------------------|
| complete(): Boolean                                 | `Job`을 완료시키며 모든 자식 코루틴이 끝날 때까지 계속 실행됩니다. 그러나 모든 코루틴이 완료된 후 이 `Job`에서 새로운 코루틴을 시작할 수 없습니다. |
| completeExceptionally(exception: Thrwable): Boolean | 주어진 예외와 함께 `Job`을 완료시킵니다. 이로 인해 모든 자식 코루틴은 즉시 취소되며 `CancellationException`이 주어진 예외를 감쌉니다. |

- 위 2가지 메서드의 결과값이 `true`인 경우 `Job`이 완료됨을 의미하며, `false`인 경우 이미 완료된 `Job`임을 의미합니다.
- `job.complete()` 호출 시 `Job` 내에 실행 중인 모든 자식 코루틴들이 완료될 떄까지 계속 실행하게 됩니다. (추가적으로 `job.join`을 사용하여 `Job`의 완료를 대기할 수 있습니다.)

### Summary

- `Job`은 코루틴을 취소하거나 상태를 추적하는 등의 기능을 통해 코루틴을 제어할 수 있습니다.
- `Job`은 자동으로 부모 코루틴에서 자식 코루틴으로 상속되지 않은 유일한 코루틴 컨텍스트 입니다.
- `job.join()` : 호출한 코루틴을 일시 중단시키고 지정된 `Job`이 완료될 때까지 대기시킬 수 있습니다.
- `job.complete()` : `Job`을 완료시키며 모든 자식 코루틴이 끝날때 까지 계속 실행됩니다.
- `job.completeExceptionally()` : 주어진 예외와 함께 `Job`을 완료시킵니다.

## [Part 2.2 : Cancellation](Cancellation.md)

### Basic Cancellation

`Job` 인터페이스에는 `cancel()` 메서드를 제공하여 작업을 취소할 수 있는 기능을 제공하며 다음 효과가 발생됩니다.

- 코루틴이 일시 정지되는 지점이 있는 경우 그 지점에서 작업이 종료합니다. (`delay` or `yield`)
- 자식 작업이 존재하는 경우 그 작업도 취소합니다. (A-B-C 작업 중 B가 취소되면 B,C가 취소되고 A는 아무런 영향을 받지 않음)
- 작업이 취소되면 `CANCELLING` 상태가 되고, 그 다음 `CANCELLED` 상태가 됩니다. (취소된 작업은 더 이상 새로운 코루틴의 부모로서 사용되지 못함)

`CancellationException` 하위 타입을 `cancel`의 인자로 전달하여 코루틴의 취소 원인을 명확하게 할 수 있습니다.  
만약 `CacnellationException` 외 타입으로 코루틴 취소 시도 시 코루틴은 취소되지 않습니다.

코루틴 취소 후 `join` 호출 시 취소가 완전히 처리된 뒤 다음 작업을 수행할 수 있습니다.
이렇게 처리되지 않으면 코루틴이 취소되기 전 다른 코드가 실행될 가능성이 있으며, 이로 인해 예기치 못한 결과 혹은 버그가 발생될 수 있습니다.

`cancel`, `join`을 같이 호출하는 `cancelAndJoin` 확장 함수가 존재합니다.

### How does cancellation work

`Job`이 취소되면 `CANCELLING` 상태로 변경되며 이 상태에서 코루틴이 다음 suspension point에 도달하면 `CancellationException`이 발생됩니다.  
`CancellationException`은 `catch` 블록을 통해 잡을 수 있으며, 이 예외를 다시 던져야 코루틴의 취소 상태가 정상적으로 상위에 전파됩니다.

취소된 코루틴은 `CancellationException`이 발생됨을 알 수 있는데 이는 코루틴이 강제로 종료되는 것이 아닌, 정상적인 예외 처리 흐름을 따르게 하여 리소스 정리를 안전하게 정리할 수 있는 기회를
줍니다.
코루틴이 취소되거나 예외가 발생하더라도 `finally` 블록이 항상 실행되는데 이 블록에서 리소스 해제나 필요한 정리 작업을 수행하면 됩니다.

### Just one more call

`CancellationException`이 발생한 후 `finally` 블록에서는 기본적으로 모든 리소스를 정리할 수 있지만,
새로운 코루틴을 시작하거나 일시 정지 함수를 호출하는 것은 허용되지 않습니다.
이는 해당 `Job`이 이미 `CANCELLING` 상태로 전환되었기 때문입니다.

그러나 `withContext(NonCancellable)` 함수를 이용하면, 코루틴이 이미 취소된 상태에서도 데이터베이스 롤백과 같은 일시 정지 함수를 안전하게 사용할 수 있습니다.
`withContext`는 실행 컨텍스트를 임시로 변경하며, `NonCancellable`은 취소할 수 없는 `Job`을 제공해 블록 내의 작업을 `ACTIVE` 상태로 유지합니다.

### invokeOnCompletion

`invokeOnCompletion` 함수는 코루틴 `Job`이 최종 상태에 도달 시 실행할 핸들러를 설정하는데 사용되고 리소스 해제, 다른 코루틴에게 알림을 보내는 등의 작업에 유용합니다.

`invokeOnCompletion`은 예외 파라미터를 전달하는데 이 값을 통해 코루틴이 어떻게 종료되는지 확인할 수 있습니다.

- `null` : 코루틴이 정상적으로 종료
- `CancellationException` : 코루틴이 취소됨
- 그 외 예외 : 코루틴이 예외와 함께 종료됨

또한 `invokeOnCompletion` 호출 전 코루틴이 완료된 경우 핸들러가 즉시 실행됩니다.

코루틴이 취소되는 순간 동기적으로 `invokeOnCompletion`을 호출하며 이는 다른 스레드에서도 실행될 수 있습니다.
즉 실행되는 스레드를 직접 제어할 수 없습니다.

### Stopping the unstoppable

취소(Cancellation)는 suspension point에서 발생되며 이러한 지점이 없는 코루틴의 경우 취소가 발생되지 않습니다.

suspension point가 존재하지 않는 코루틴을 취소하는 방법으로는 다음과 같이 있습니다.

- `yield()`를 호출하여 코루틴을 잠깐 중단하고 다시 재개하여 코루틴을 취소할 수 있는 suspension point를 생성하는 방법
- `Job.isActive` 프로퍼티를 사용하여 현재 코루틴이 `ACTIVE` 상태인지 확인 후 적절한 조치를 취하는 방법
- `ensureActive()`를 사용하여 `Job`이 `ACTIVE` 상태가 아닌 경우 `CancellationException`을 발생시키는 방법

`yield()`는 일반적인 최상위 suspension 함수이며, 스레드 변경 등 다른 효과를 가질 수 있습니다.
`ensureActive()`는 특정 `coroutineScope`나 `Job`에서만 호출될 수 있으며, 코루틴이 `ACTIVE` 상태가 아닌 경우에만 예외를 발생시킵니다.

### suspendCancellableCoroutine

`suspendCancellableCoroutine` 함수는 코루틴 내에서 비동기 작업을 수행할 떄 사용됩니다.
표준 `suspendCoroutine`과 달리 취소 가능성을 고려한 추가적인 메서드들을 제공합니다.

`invokeOnCancellation`은 코루틴이 취소되었을 때 실행될 코드 블록을 정의하는 메서드로 불필요한 리소스 해제와 실행 중인 작업 중단 등을 실행할 수 있습니다.

### Summary

- `Job` 인터페이스는 코루틴을 취소할 수 있는 `cancel()`을 제공하며 코루틴이 취소된 경우 자식 `Job`도 함께 취소됩니다.
- 코루틴이 취소되면 다음 'suspension point'에서 `CancellationException`이 발생되며 `finally` 블록에서 리소스를 정리하는 것이 좋습니다.
- `withContext(NonCancellable)`을 사용하면 이미 취소된 코루틴에서도 suspension 함수를 호출할 수 있습니다.
- `invokeOnCompletion` 함수는 '코루틴 종료' 시 어떤 작업을 실행할 지 설정할 수 있으며, 코루틴이 어떤 예외로 종료되는지 확인할 수 있습니다.
- `suspendCancellableCoroutine` 함수는 코루틴 내 비동기 작업을 수행할 때 사용되며 취소 가능성을 고려한 추가적인 메서드들을 제공합니다.
    - `invokeOnCancellation` 함수는 '코루틴 취소' 시 어떤 작업을 실행할 지 설정 할 수 있습니다.

---

## [Part 2.4 :Exception handling](Exception%20handling.md)

### Stop breaking my coroutines

`SupervisorJob`은 일반 `Job`과 다르게 자식 코루틴에서 발생된 예외가 부모 코루틴에 영향을 주지 않습니다.  
즉, 자식 코루틴에서 예외가 발생하더라도 다른 자식 코루틴이나 부모 코루틴이 중단되지 않습니다.

일반적으로 `SupervisorJob`은 아래와 같이 여러 코루틴을 시작하는 `Scope`의 일부로 사용되며 해당 `Scope`에서 시작되는 모든 코루틴은 자동으로 `SupervisorJob`에 의해 관리됩니다.

```kotlin
val scope = CoroutineScope(SupervisorJob())

scope.launch { ... }
scope.async { ... }
```

자주 범하는 실수 중 하나는 `launch`, `async`의 인자로 `SupervisorJob`을 사용하는 것인데,
이는 오직 하나의 직접적인 자식만을 가지고 있기에 일반 `Job`을 사용하는 것과 같이 예외 처리에 큰 이점을 가질 수 없습니다.  
이처럼 큰 이점을 가지려면 `SupervisorJob`은 여러 코루틴 빌더의 컨텍스트로 사용하는 것이 더 효과적입니다.

```kotlin
val job = SupervisorJob()

launch(job) { ... }
async(job) { ... }
job.cancelAndJoin()
```

`supervisorScope`는 예외 전파를 중단하는 또 다른 방법으로 부모 코루틴과의 연결을 유지하면서도 자식 코루틴에서 발생하는 예외를 무시하거나 제어 할 수 있습니다.

`coroutineScope`는 `launch`, `async` 등의 코루틴 빌더와 다르게 부모 코루틴에게 예외를 전파하지 않고 해당 Scope 내에서 예외를 잡을 수 있게 해줍니다.
코루틴 빌더는 자체적으로 예외를 잡지 않기에, 부모 코루틴에게 예외가 전파될 가능성이 높습니다.

`supervisorScope`는 자식 코루틴의 예외가 부모 코루틴에게 전파되지 않지만,
`withContext(SupervisorJob())`에서 `SupervisorJob()`을 부모 `Job`으로 만들지만, `withContext`는 단순히 코루틴의 컨텍스트를 변경하는 것이기에 예외 무시 기능이
적용되지 않습니다.

### await

`await`은 `async` 코루틴 빌더와 함께 사용되며 비동기 작업의 결과를 가져오기 위한 메서드입니다.
다른 코루틴들과 동일하게 비동기 작업에서 예외가 발생되면, 부모 코루틴까지 예외를 전파합니다.

### CancellationException does not propagate to its parent

예외가 `CancellationException`의 하위 클래스라면, 이 예외는 부모로 전파되지 않고 현재 코루틴만 취소합니다.

### Coroutine exception handler

`CoroutineExceptionHandler`는 코루틴에서 예외 처리 시, 모든 예외에 대한 기본 동작을 정의하는데 유용합니다.
이 핸들러는 예외가 발생하면 정의한 코드 블록이 실행되며, 코루틴 컨텍스트에 추가할 수 있어 `SupervisorJob` 등과 같이 사용할 수 있습니다.

### Summary

- `SupervisorJob`에서 발생된 예외는 부모 코루틴에게 전파되지 않으며, 이로 인해 여러 코루틴 빌더의 컨텍스트로 사용하는 것이 효과적입니다.
- `suervisorScope` 블록에서 생성된 코루틴들은 예외가 발생해도 부모 코루틴에게 전파되지 않습니다.
- 예외가 `CancellationException`의 하위 클래스라면, 이 예외는 부모로 전파되지 않고 현재 코루틴만 취소합니다.
- `CoroutineExceptionHandler`는 코루틴에서 예외 처리 시, 모든 예외에 대한 기본 동작을 정의하는데 유용합니다.

---

## [Part 2.5 : Coroutine scope functions](CoroutineScope%20함수.md)

### Approaches that were used before coroutine scope functions were introduced

코루틴 빌더(`async`, `launch` 등)을 사용하려면 스코프가 필요하며 이는 `runBlocking`, `coroutineScope` 등을 통해 제공됩니다.

스코프 중 `GlobalScope`도 존재하는데 이 스코프는 다음과 같은 특징을 지닙니다.

- `GlobalScope`는 애플리케이션의 전체 수명 주기 동안 유지되며, 애플리케이션의 수명 주기와 동일하게 실행되기에 부모 코루틴이 취소되더라도 계속 실행됩니다.
- `GlobalScope`는 `EmptyCoroutineContext`를 얻기에 컨텍스트와 스코프에 대한 값이 없어 일반적인 부모-자식 간 상속이 없습니다.

이와 같은 특징으로 인해 '리소스 낭비가 발생'할 수 있으며, '컨텍스트에 대한 설정을 직접 해야하고', '유닛테스트가 어려워진다'는 단점이 있습니다.

### coroutineScope

`launch`나 `async`는 새로운 코루틴을 생성하고 즉시 다음 코드 라인으로 넘어가지만
`coroutineScope`는 자식 코루틴들이 모두 완료될 떄까지 생성된 코루틴을 일시 중단합니다.

즉, `coroutineScope`는 '일시 정지 가능한 코드 블록'을 실행하기 위한 새로운 코루틴 스코프를 생성합니다.

`coroutineScope`는 다음과 같은 특징을 지닙니다.

- 상위 스코프의 `coroutineContext`를 상속받아 생성된 자식 코루틴에게 해당 컨텍스트를 상속합니다.
- 생성된 자식 코루틴이 완료되어야지만, 자기 자신을 종료할 수 있습니다.
- `coroutineScope`가 속한 부모 코루틴이 취소된다면, 생성된 자식 코루틴들도 모두 취소됩니다.

`coroutineScope` 또는 그 안의 자식 코루틴에서 예외가 발생되면 해당 범위의 코루틴들이 모두 취소되고 부모 코루틴에 예외를 던집니다.

### Coroutine Scope functions

코루틴 스코프 함수는 suspending 함수 내에서 새로운 코루틴 스코프를 생성하거나 기존의 스코프를 변형하는데 사용됩니다.
이러한 코루틴 스코프 함수에는 `coroutineScope`, `superviorScope`, `withContext`, `withTimeout` 등이 있습니다.

코루틴 스코프 함수와 코루틴 빌더는 혼동될 수 있지만 이 둘을 엄연히 [차이점](CoroutineScope%20함수.md#코루틴-스코프-함수와-코루틴-빌더의-차이점)이 존재합니다

코루틴 스코프 함수와 `runBlocking`도 비슷해 보이지만, `runBlocking`은 blocking 함수, 코루틴 스코프 함수는 suspending 함수 입니다.
이로 인해 서로 다른 목적과 환경에서 사용되므로 명확하게 구분해야 합니다.

### withContext

컨텍스트 변경이 필요한 경우 `withContext`를 사용하는 것이 유용하며, 컨텍스트의 변경이 필요 없는 경우 `coroutineScope`를 사용하면 됩니다.

`withContext`는 코드 중간에 다른 코루틴 스코프를 설정하기 위해 `Dispatchers` 컨텍스트와 함께 자주 사용됩니다.

코드의 가독성과 관리성을 위해서 `async`와 `await()`를 즉시 사용할 떄 특별한 경우가 아니라면 `coroutineScope` or `withContext`를 사용하게 좋습니다.

### supervisorScope

`coroutineScope`와 유사하지만, 컨텍스트의 `Job`을 `SupervisorJob`으로 오버라이드하여 자식 코루틴이 예외를 발생시켜도 다른 코루틴들을 취소 시키지 않습니다.
이 때문에 여러 독립적인 비동기 작업을 안전하고 효율적으로 관리하는 데 주로 사용됩니다.

`supervisorScope` 내부에서 `async`와 `await()`를 사용하여 코루틴의 결과를 얻는 중 예외가 발생되면 `await()` 기준으로 예외가 전파되기에,
이를 고려하여 `await()` 호출 주변에 `try-catch`와 같은 예외 처리 코드를 작성해야 합니다.

`withContext(SupervisorJob())` 사용 시, `SupervisorJob()`은 `withContext`의 `Job`에 부모로 사용되게 됩니다.
이 떄문에 `withContext`의 자식 코루틴에서 예외가 발생하게 되면 `withContext`로 전파되기에 `SupervisorJob()`은 크게 의미가 없습니다.

### withTimeout

`withTimeout`은 작업에 최대 실행 시간을 부여하고 다음과 같이 처리됩니다.

- 시간이 초과되면 `TimeoutCancellationException`을 발생시켜 작업을 강제로 종료시킵니다.
- 성공적으로 완료된 경우 블록에서 반환된 값을 얻습니다.

주로 네트워크 요청, 복잡한 알고리즘 등의 처리 시간에 민감한 곳에 사용될 수 있으며, 이는 테스트에 유용하게 사용될 수 있습니다.

`withTimeoutOrNull`은 `withTimeout`과 동일하지만, 시간 초과 시 `TimeoutCancellationException`을 발생시키지 않고 `null`을 반환합니다.
이로 인해 좀 더 유연한 예외 처리를 할 수 있습니다.

### Connecting coroutine scope functions

코루틴 스코프 함수들은 자체적으로 특정한 기능을 제공하며 이 함수들을 중첩해서 사용하여 여러 기능을 조합해서 사용할 수 있습니다.

단, 각 함수의 특정과 영향성을 잘 이해하고 사용해야 합니다.   
(`withContext`가 취소 불가능한 작업을 수행하는데 `withTimeout`은 의미 없는 기능이 될 수 있습니다.)

### Additional operations

기본적으로 suspending 함수는 내부 코루틴 작업이 모두 완료될 때까지 완료되지 않습니다.
이때 같은 코루틴 스코프에서 별도의 추가 작업(Analytics 호출 등)과 같은 작업이 포함되어 있는 경우에
추가 작업에서 예외가 발생되면 메인 로직이 취소 되기에 좋은 방식이 아닙니다.

이 때문에 명시적으로 스코프를 주입하여 사용하는 것이 좋습니다.
명시적으로 스코프를 주입하면 해당 로직이 독립적으로 비동기 작업을 할 수 있음이 명확해지고, 해당 코루틴 작업의 흐름을 쉽게 파악할 수 있어 디버깅이 간단해집니다. 

### Summary

- 코루틴 스코프 함수를 통해 코루틴 빌더들을 사용할 수 있으며, 코루틴 스코프 함수 중 `GlobalScope`는 다음과 같은 특징을 지닙니다.
  - Application Lifecycle을 가지며 Application 종료될 떄까지 해당 스코프에서 실행된 코루틴 작업이 실행 됩니다.
  - `EmptyCoroutineScope`를 통해 생성되므로 컨텍스트와 스코프에 대한 값이 없습니다.
- `coroutineScope`는 '일시 정지 가능한 코드 블록'을 실행하기 위한 새로운 코루틴 스코프를 생성하며 다음 특징을 지닙니다.
  - 상위 스코프의 컨텍스트를 상속받아 자식 코루틴에게 전달합니다.
  - 자식 코루틴이 완료되어야지만, 자신도 종료할 수 있습니다.
  - 부모 코루틴이 취소되면 자기 자신을 포함한 모든 자식 코루틴도 취소됩니다.
- 코루틴 스코프 함수에는 `coroutineScope`, `withContext`, `superviorScope`, `withTimeout` 등이 있습니다.
  - `withContext` : `Dispatchers`의 변경과 같이 코루틴 컨텍스트의 변경이 필요한 경우 사용됩니다.
  - `supervisorScope` : `SupervisorJob`을 사용하여 자식 코루틴의 예외를 전파하지 않는 독립적인 비동기 작업에 사용됩니다.
  - `withTimeout` : 특정 작업에 시간 제한을 두고 싶을 때 사용됩니다.
- 코루틴 스코프 함수들은 중첩해서 여러 기능을 조합해서 사용할 수 있습니다.
- 코루틴 내부에서 메인 로직과 추가 작업을 같이 실행 시, 별도의 스코프를 생성하여 클래스에 명시적으로 주입한 뒤 별도의 스코프로 추가 작업을 실행하는 것이 메인 로직에 영향을 주지 않기에 권장됩니다.

---

## [Part 2.6 : Dispatchers](Dispatchers.md)

### Default dispatcher

코루틴 빌더에서 디스패처를 지정하지 않으면 기본적으로 `Dispatchers.Default`가 선택되며 이 디스패처는 다음 특징을 지닙니다.

- CPU 집약적인 작업을 수행하기 위해 최적화 되어 있으며, 스레드 풀의 크기는 실행 환경의 CPU 코어 수에 따라 결정됩니다.
- 블로킹 연산을 수행할 경우, 스레드가 대기 상태에 빠져 자원을 낭비할 수 있으므로 블로킹 연산에 해당 디스패처는 적절하지 않습니다.

`runBlocking`은 다른 디스패처가 설정되지 않은 경우 기본적으로 `Dispatchers.Default`가 아닌 메인 스레드에서 실행됩니다.

### Limiting the default dispatcher

비용이 많이 드는 연산에서 디스패처에 `limitedParallelism`를 적용하면 사용할 수 있는 스레드 수를 제한할 수 있습니다.

이 메커니즘은 `Dispatchers.IO`에서 사용하는 것을 주된 목적으로 만들어졌지만, 다른 디스패처에서도 사용할 수 있습니다.