Rxjava, Reactor와 같이 잘 정리된 기존 JVM 라이브러리들이 있습니다. 또한 Java 자체도 멀티스레딩을 지원하며, 많은 개발자들이 단순한 콜백을 사용하기도 합니다.
이처럼 명백하게 비동기 작업을 위한 여러가지 옵션들이 있습니다.

그러나 코루틴은 그 이상을 제공합니다.
코루틴은 멀티플랫폼으로 모든 Kotlin 플랫폼(JVM, JS, IOS, 공통 모듈 등)에서 사용될 수 있습니다.
또한 코드 구조를 크게 바꾸지 않습니다. 대부분의 코루틴 기능은 간단하게 사용할 수 있습니다.

## [Part 1.1 : Why Kotlin Coroutines](왜%20코루틴%20인가%3F.md)

> - Android에서 'View Update'는 'MainThread'에서만 가능, 만약 'MainThread' 차단 시 앱 크래쉬 발생
> - 이를 위해 'Thread switching', 'Callback', 'Reactive Programming' 등 제시되었으나, 각각의 문제점 존재
>   - Thread switching : 취소 메커니즘이 없고, Thread 생성은 비용이 많이 들고 Thread Switcthing의 관리가 어려움 
>   - Callback : 자연스러운 병렬 처리 미 지원, 취소 메커니즘 구현 시 추가적인 구현이 필요, 'Callback Hell' 발생 가능
>   - Reactive Programming : 코드 복잡성 증가, 명시적인 취소가 반드시 필요
> - Coroutine 도입 핵심 기능은 'Coroutine을 특정 시점에 중단 후 재개'

클라이언트 로직 중 빈번하게 처리하는 작업의 순서는 다음과 같습니다.

1. 하나 or 여러 데이터 소스(API, Database, Preference, 다른 앱 등)에서 데이터를 얻어옴
2. 데이터 처리 (파싱, 추출, 병합 등)
3. 처리된 데이터를 통해 View Update, DB Update, API 요청 등 작업 수행

여기서, 'ViewUpdate'는 Android 플랫폼에서 'MainThread'에서만 가능합니다.  
만약 'MainThread'에서 'Blocking' 작업을 하는 경우 앱이 멈출 수 있으며, 이는 앱의 크래쉬로 이어질 수 있습니다.

따라서 'Blocking' 작업을 하는 경우, 'Worker-Thread'를 통해 처리하는 것이 일반적인 관행이었습니다.  
그러나 'Thread switching'의 경우에도 문제점이 나타나게 됩니다.

1. Thread 취소 메커니즘이 없어, 빈번한 메모리 누수 현상
2. 많은 Thread 생성은 많은 비용이 발생
3. 빈번한 Thread switching은 관리가 어려움
4. 코드가 복잡해짐

위 문제를 해결하기 위한 다른 패턴으로 'Callback'이 제시되었습니다.  
'Callback' : 함수를 'non-blocking'으로 만들되, 'Callback' 프로세스가 완료되면 실행해야 하는 함수를 전달하는 것입니다.

'Callback'에서도 문제점이 나타나게 되는데 이는 다음과 같습니다.

1. 자연스러운 병렬 처리 미 지원
2. 'Callback'의 취소를 위해, 이들을 추적하고 관리하는 추가적인 코드가 필요함
3. 'Callback'의 중첩으로 인해, 'Callback Hell'이 발생할 수 있음
4. 프로그램 진행 순서를 제어하기 어려움

이를 위해 'Reactive Programming'이 제시되었으며, 이는 모든 '작업'의 시작과 처리를 'Observable DataStream' 내에서 이루어집니다.  
이런 'DataStream'은 'Thread Switching'과 'Concurrency Processing'을 지원하여 앱에서 '작업'을 병렬로 수행하는데 자주 활용됩니다. 

'Reactive Programming' 방식은 메모리 누수가 없고, 자연스러운 취소를 지원하며, 'Thread'를 효율적으로 사용하기에 'Callback' 보다 더 좋은 방법이 됩니다.  
하지만, 다음과 같은 문제점들이 생길 수 있습니다. 

1. '코드 복잡성' 커질 수 있음
2. 'DataStream'의 취소가 반드시 명시적으로 이루어져야 함
3. 'Reactive Programming' 도입 시 코드 상당 부분의 리팩토링 필요

이를 위해 'Coroutine'이 만들어지게 됩니다.

Coroutine 도입의 핵심 기능은 **Coroutine을 특정 시점에 중단 후 재개** 입니다.  
이 덕분에 'MainThread'에서 Blocking 작업 시 Thread Blocking이 아닌, 'Coroutine 중단'이 가능해집니다.  
Coroutine이 '중단'되면, 'Thread'는 'Blocking' 되지 않고, View Update 또는 다른 Coroutine을 처리하는 데 사용할 수 있습니다.

특정 작업을 호출하여 데이터를 얻는 Coroutine이 중단 후 '데이터가 준비'되면 Coroutine은 'MainThread'를 기다립니다.  
기다린 후 'MainThread'에서 Coroutine이 재개되면, '중단한 지점'부터 프로세스를 다시 진행할 수 있습니다.

## [Part 1.2 : Sequence builder](시퀀스%20빌더.md)

> - 'Sequence'는 요청이 있을 때마다 요소를 생성하는 연산을 수행, 만약 여러 요소를 포함하는 'Sequence'는 한 번 요소를 출력한 후, 해당 위치를 기억하고, 다음 요소 요청이 발생되면 이전에 '중단 위치'에서부터 연산을 재개하여 다음 요소를 출력
> - 이러한 과정을 '일시 정지 메커니즘'이라 하며, 이는 코루틴의 작동 방식인 '중단과 재개'와 유사한 개념

Kotlin은 'Generator' 대신 'Sequence Builder'를 제공하여 'Sequence'를 생성하여 사용합니다.  
'Sequence'는 'Collection'과 유사한 개념이지만, '요소'가 필요할 때 계산되는 'Lazy'한 특징을 가집니다.  
그 결과, 'Sequence'는 다음 특징을 지닙니다.

1. '요소'의 요청에 의해서만 연산되므로 '최소한의 작업을 처리'함
2. 미리 모든 '요소'를 생성하지 않기에 '메모리 효율적'
3. 계속해서 '요소'를 생성하는 것이 가능하여 '무한'할 수 있음

'Collection'과 'Sequence'의 중요한 차이점은 'Collection'은 '모든 요소'를 미리 생성하는 반면, 'Sequence'는 '요소'가 필요하다는 요청에 의해서만 연산됩니다.

요소가 필요할 때 계산된다는 것은 'Sequence'가 이전 요소를 출력 후 '중단된 시점으로 이동'하여 '다음 요소를 출력'하는 것입니다. 
일반 함수의 경우 '중단'된 후 '중단된 시점'으로 돌아가 다음 작업을 수행할 수 없기에 이러한 '일시 정지 메커니즘'은 'Sequence'에서 중요한 메커니즘으로 작용합니다.

## [Part 1.3 : How does suspension work?](코루틴에서%20일시정지는%20어떻게%20동작될까%3F.md)

> - Coroutine 중단 시 `Continuation`이 발생되고, 이를 통해 다시 '재개'할 수 있음 
> - `Continuation`의 `resume()`, `resumeWithException()`으로 Coroutine 재개 시 '결과 또는 예외'를 얻음
> - `dealy`, `suspendCoroutine`와 같은 호출은 'suspend function' 중단 X, Coroutine의 중단 O

Thread는 작업을 일시 중단하고 저장하는 기능이 없으며, 작업이 중단될 때에는 'Blocking' 상태가 됩니다.  
반면, Coroutine은 '중단' 상태일 때 자원을 사용하지 않으며, `Continuation` 객체를 통해 다른 Thread에서 Coroutine을 '재개'할 수 있습니다.

Coroutine은 `suspendCoroutine<T>`을 통해 '중단'될 수 있고, `Continuation`의 `resume(value: T)`을 통해 다시 '재개'할 수 있습니다.
이 때 `resume(value: T)`에 전달되는 값은 `suspendCoroutine<T>`의 반환 타입과 일치해야 합니다.  

추가로 Coroutine 재개 시 '결과' 또는 '예외' 전달이 필요한 경우, 
`suspendCancelableCoroutine<T>`과 `resumeWithException(exception: Throwable)`을 통해 처리가 가능합니다.

주의할 점으로 `suspendCoroutine<T>`을 통한 '중단 상황'은 **'suspend function'의 중단이 아닌, Coroutine의 중단**입니다.  

## [Part 1.4 : Coroutines under the hood](코루틴%20내부%20동작.md)

> - `Continuation`은 상태를 저장하며 이를 통해 Coroutine은 함수의 정확한 지점에서 재개가 가능
>   - label : 'suspending function'이 중단된 후 재개되는 지점
>   - local data : 'suspending function'의 파라미터 혹은 로컬 변수
>   - parent continuation : 'suspending function'을 호출한 상위 `Continuation`
> - `Continuation`은 자신을 호출한 'Parent Continuation'과 연결되어 'Continuation Chain'을 형성하며, 이는 'Call Stack'과 유사한 역할을 함
> - CPS : `Continuation`을 마지막 인수로 함수에서 함수로 전달, 이를 통해 'Continuation Chain' 형성 
> - Coroutine이 중단 상태일 경우 `COROUTINE_SUSPENDED`를 'Call Stack' 최상단까지 전파하여, Thread를 다른 작업에 사용 가능하도록 알릴 수 있음
> - Continuation 재개 시 발생하는 과정
>   1. 각 `Continuation`은 자신과 연결된 함수 호출
>   2. 자신과 연결된 함수 작업 완료 시, 해당 `Continuation`은 자신을 호출한 `Continuation` 재개
>   3. 이어서, 호출된 `Continuation`과 연결된 함수 호출, 위 과정을 반복
>   4. 이러한 반복을 'Call Stack' 최상단에 도달할 때까지 계속 진행

'suspending function'은 실행 중 중단되었다가 나중에 재개될 수 있는 함수입니다.  
이 함수는 'suspend point'에서 실행을 중단할 수 있으며, 이를 통해 함수는 '여러 상태'를 가질 수 있습니다.

각 상태는 고유 '식별 번호'를 가지며, 함수의 '로컬 데이터'와 함께 `Continuation` 객체에 저장됩니다.   
이렇게 저장된 정보는 함수 재개 시 필요한 모든 컨텍스트를 제공합니다.

`Continuation` 객체는 자신을 호출한 '상위 suspending function'의 `Continuation` 객체와 연결됩니다.   
이들은 'Continuation Chain'을 형성하고, 이는 전체 프로그램의 'Call Stack'과 유사한 역할을 합니다.

'Continuation Chain'은 Coroutine이 중단 후 재개 시, 
함수 실행을 정확한 지점에서 이어갈 수 있도록 하고, 이전에 중단된 상태를 복원하는 데 사용됩니다.

---

'CPS(Continuation-passing-style)'는 `Continuation`을 인수로, 함수에서 함수로 전달되는 것을 의미하며 마지막 파라미터에 놓입니다.

```kotlin
suspend fun getUser(): User
suspend fun setUser(user: User)

// under the hood is
fun getUser(continuation: Continuation<*>): Any?
fun setUser(user: User, continuation: Continuation<*>): Any

// Intrinsics.kt
val COROUTINE_SUSPENDED: Any get() = CoroutineSingletons.COROUTINE_SUSPENDED
```

'suspending function'이 변환된 코드를 보면, 반환 타입이 `Any?` 또는 `Any`로 변경됨을 알 수 있습니다.  

이는 '중단'되어 선언된 타입으로 반환 되지 않을 수 있기에, 이런 경우 `CORUTINE_SUSPENDED`를 반환할 수 있게 됩니다. 
그 결과 `getUser`의 반환 타입은 `User?`와 `Any`의 가장 가까운 상위 타입인 `Any?`가 됩니다.

---

Coroutine은 `Continuation`을 통해 함수의 정확한 지점으로 재개됩니다.

이는 `Continuation`의 `label` 속성을 통해 'suspending function'의 재개 위치를 저장합니다.  
`label`의 초기값은 `0`(함수 첫 시작 위치)으로 그 외 숫자들은 '중단 후 다시 재개되는 지점'을 의미합니다.

만약 Coroutine이 중단되면, `COROUTINE_SUSPENDED`를 반환하고, 이를 'Call Stack' 최상단까지 전파되어 'Coroutine 빌더' 또는 `resume` 함수에 도달할 때 까지 이어집니다.
이러한 동작으로 인해, Thread가 다른 작업(다른 coroutine)에 사용 가능하도록 처리됩니다.

---

Coroutine 중단 후 재개할 때 '로컬 데이터' 상태가 있다면, `Continuation`에 반드시 저장되어야 합니다.  
'로컬 데이터'는 '중단 직전'에 `Continuation`에 '저장'되며, '재개' 시 데이터는 함수 시작 부분에서 '복원'됩니다.

'suspending function'의 반환 값으로 '재개' 되는 경우에도 `Continuation`으로 관리해야 합니다.  
이 경우 2가지로 나뉘는데, 반환이 값이라면 `Result.Success(value)`를 통해 재개되고, 반환이 예외라면 `Result.Failure(exception)`을 통해 재개됩니다. 

---

`Continuation`은 다음 3가지 정보를 저장합니다.

- 고유 식별 번호(`label`)
- 자신의 '로컬 데이터'
- 자신을 호출한 '상위 suspending function'의 `Continuation`

이처럼 상위 `Continuation`은 다른 `Continuation`을 참조하는 형식으로 'Continuation Chain'을 형성합니다. 
결과적으로 이는 'Call Stack'과 유사한 역할을 할 수 있게 됩니다.

`Continuation`이 '재개'될 때 발생하는 과정은 다음과 같습니다.

1. 각 `Continuation`은 자신과 연결된 함수 호출
2. 자신과 연결된 함수 작업 완료 시, 해당 `Continuation`은 자신을 호출한 `Continuation` 재개
3. 이어서, 호출된 `Continuation`과 연결된 함수 호출, 위 과정을 반복
4. 이러한 반복을 'Call Stack' 최상단에 도달할 때까지 계속 진행

예외 처리도 이와 비슷한 메커니즘을 따릅니다.

```kotlin
fun printUser(
    continuation: Continuation<*>
) {
    // ...
    if(continuation.label == 1) {
        continuation.label = 2 // setting next label
        continuation.userId = userId // Storing state on continuation
        val res = getUserName(userId, continuation) // Calling suspending function
        if (res == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED // Suspension
        result = Result.success(res) // Setting result if not suspended
    }

    if(continuation.label == 2) {
        result!!.throwOnFailure() // Throwing is resumed with exception
        userName = result.getOrNull() as String // Reading result value
        // ...
    }
}

class PrintUserContinuation(
    val completion: Continuation<Unit>
): Continuation<Unit> {
    // ...
    
    override fun reusmeWith(result: Result<Any>) {
        // ...
        
        val res = try {
            val r = printUser(this)
            if(r == COROUTINE_SUSPENDED) return
            Result.success(r as Unit)
        } catch (e : Throwable) {
            Result.failure(e)
        }
        
        completion.resumeWith(res)
    }
}

```

처리되지 않은 예외는 `resumeWith()`에서 포착되고, 그 후에 `Result.failure(e)`로 감싸져, 자신을 호출한 함수는 이 결과와 함께 재개 됩니다.

---

일반 함수 대비, 'suspending function'을 사용함으로 발생하는 비용은 다음 같이 크지 않습니다.

- 'suspending function'을 여러 상태로 나누는 작업은 단순한 번호(`label`) 비교와 실행 점프로 인한 비용이 거의 들지 않음
- `Continuation`에 '상태'를 저장하는 것도 로컬 변수를 복사하는 것이 아닌, 새로운 변수가 같은 메모리 주소를 가리키도록 만드는 것이기에 비용이 적음
- `ContinuationClass`는 '중단과 재개'가 가능한 함수의 상태를 관리하기 위해 필요하며, 이는 1번만 생성되기에 전체적인 성능에 미치는 영향은 미미함

---

## [Part 1.5 : Coroutine: Built-in support vs library](코루틴의%20구조%20지원%20vs%20라이브러리.md)

Coroutine은 'Built-in Support'와 'kotlinx.coroutine library' 구성요소로 이루어져 있습니다.

'Built-in Support'는 Kotlin 언어 자체에서 제공하는 기능으로 매우 기본적이고 유연합니다.  
그러나 이러한 유연성으로 인해 직접 사용하기 복잡하며 일반적으로 오픈소스 라이브러리 개발자들이 사용합니다.

'kotlinx.coroutine library'는 프로젝트에 별도로 추가하며, 'Built-in Support' 보다 더 높은 수준의 추상화를 제공합니다. 
일반 개발자들이 사용하기 간편하고 편리하게 동시성을 다룰 수 있는 구체적인 방법을 제공합니다.