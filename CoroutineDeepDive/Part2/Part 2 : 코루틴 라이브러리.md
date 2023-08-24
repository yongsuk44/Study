# Kotlin Coroutines library

코루틴을 제대로 사용하기 위해 필요한 모든 것을 배우며 아래의 항목들을 정리하는 파트 입니다.

- 코루틴 빌더
- 다양한 코루틴 컨텍스트
- 코루틴 취소 작동 방식 
- 코루틴 설정 방법
- 코루틴 테스트 방법
- 공유 상태에 안전하게 접근하는 방법

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
이러한 특징은 `Thread.sleep(1000L)`과 `delay(1000L)`이 동일하게 동작하게 하므로, 메인 함수나 단위 테스트와 같이 프로그램이나 테스트가 너무 일찍 종료되는 것을 방지하고자 할 때 유용합니다.

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
  - `defaultContext`에는 기본 설정들이 존재하며 `ContinuationInterceptor` 값이 없으면 `Dispatchers.Default` 적용되며, 디버그 모드 시 `CoruotineId`를 적용 합니다.
- `Job`은 코루틴 생명주기를 나타내는 특별한 컨텍스트 요소로, 부모-자식 간의 통신에 사용되는 Mutable한 컨텍스트입니다.