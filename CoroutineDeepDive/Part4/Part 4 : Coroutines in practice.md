## [Part 4.1 : Common use cases](Common%20use%20cases.md)

대부분 Application 구조는 다음 3가지 계층으로 나눕니다.

- Data / Adapter Layer
- Domain Layer
- Presentation / UI Layer

### Data / Adapter Layer

데이터 저장, 추출, 변환 등 작업을 합니다.   
즉, 네트워크 요청, 데이터베이스 작업 등을 하는 영역입니다.

여기서 코루틴은 `suspend` 키워드를 통해 각 작업을 비동기적으로 수행할 수 있게 지원해줍니다.  
추가로, `Room` 라이브러리의 경우 `Flow`를 지원하며 쉬운 반응형 UI 구현을 도와줍니다.

#### callback functions

만약 코루틴을 지원하지 않고 콜백 함수를 사용하도록 강제하는 라이브러리를 사용해야 하는 경우,
`suspendCancellableCoroutine`을 사용하여 콜백 함수를 일시 중지 함수로 변환할 수 있습니다.

콜백 호출 시 `resume`을 통해 코루틴을 재개할 수 있으며, 콜백이 취소 가능한 경우 `invokeOnCancellation`을 통해 취소 로직을 정의할 수 있습니다.
또한 콜백 호출의 성공과 예외를 명확하게 해야하는 경우 `resume`과 `resumeWithException`을 사용할 수 있습니다.

#### Blocking functions

코루틴 내에서 블로킹 함수 호출 시 스레드가 차단되어 해당 스레드는 다른 코루틴 작업에 사용할 수 없게 됩니다.  
이처럼 코루틴에서 블로킹 작업이 필요한 경우 `withContext`와 `Dispatchers`를 지정해서 사용할 수 있습니다.

만약 디스패처에서 사용되는 스레드 풀의 크기를 조절하고 싶으면 `limitedParallelism`을 통해 조절할 수 있습니다.  
이를 통해 스레드의 수를 제한하거나 늘려 특정 작업에 맞는 사용자 정의 디스패처를 만들 수 있습니다. 

#### Observing with Flow

일시 중지 함수는 단일 값을 생성하거나 가져오는데 적합하지만, 여러 값을 생성하거나 가져올 땐 `Flow`가 적절합니다.

`Flow`는 다음과 같은 상황에서 사용될 수 있습니다.

- `Room`에서 테이블의 변경사항을 관찰할 때
- 실시간 메시지 수신
- 버튼 클릭이나 텍스트 변경과 같은 UI 이벤트를 관찰할 때

### Domain Layer

도메인 계층은 비지니스 로직이 구현되는 곳으로 Usecase, Services 등을 정의합니다.  
또한 도메인 계층에서는 코루틴 스코프 객체에 직접 작업을 수행하거나, suspend 함수를 외부에 노출하는 것을 피해야 합니다.

실제로 도메인 계층에서 비지니스 로직 구현 시 여러 작업을 연속으로 수행하는 경우가 많기에 
다른 suspend 함수를 호출하는 suspend 함수를 주로 가지고 있습니다.

#### Concurrent calls

병렬로 비동기 작업을 실행하려면, 하나의 코루틴 스코프 안에 `async` 빌더를 사용하여 시작할 수 있습니다.

이 떄 컬렉션 함수와 함께 사용하여 리스트의 각 요소에 대한 비동기 작업 시 
`awaitAll`을 통해 모든 비동기 작업의 완료를 기다리고 모든 작업이 완료되면 그 결과를 한 번에 가져올 수 있습니다.

만약 컬렉션을 통해 여러 비동기 호출 시 동시 호출 수를 제한하려면 
`Flow`로 변환하여 `flatMapMerge`의 `concurrency` 파라미터를 통해 동시 호출의 수를 제한할 수 있습니다.

코루틴은 동시성 구조화로 인해 자식 코루틴에서 예외가 발생하면 전체 코루틴을 중단시키는 점을 주의해야 합니다.
이 때 `supervisorScope`를 사용하면 여러 코루틴 작업을 독립적으로 수행되어 하나의 코루틴이 취소되어도 전체 코루틴에 영향을 주지 않습니다.

`withTimeout` 또는 `withTimeoutOrNull`을 사용하면 작업 실행 시간을 제한하고 지정된 시간을 초과하면 작업을 자동으로 취소합니다.

---

### Presentation / UI Layer

코루틴은 주로 이 계층에서 실행됩니다.

안드로이드에서는 WorkManager를 통해 백그라운드 작업을 쉽게 관리하고 예약할 수 있습니다.
이떄 코루틴은 `CoroutineWorker`를 제공하여 자동으로 코루틴에서 실행하기에 별도로 코루틴을 시작할 필요가 없습니다.

이처럼 특정 라이브러리는 코루틴을 자동으로 시작해주지만, 대부분의 경우에는 그렇지 않습니다.  
이런 경우 `scope` 객체에 코루틴 빌더를 사용하여 코루틴을 시작합니다.

안드로이드에서는 `viewModelScope`, `lifecycleScope`를 통해 안드로이드의 생명주기와 쉽게 연동하여 비동기 작업을 더욱 효율적으로 관리할 수 있습니다.

#### Creating custom scope

코루틴 스코프를 직접 만들어 사용하면 코루틴의 생명주기를 더욱 세밀하게 관리할 수 있습니다.  
또한 직접 만들어 사용하면 디스패처나 예외 핸들러를 설정할 수 있어 더욱 유연하게 사용할 수 있습니다.

```kotlin
private val exceptionHandler = CoroutineExceptionHandler { _, e -> Log.e(e) }
private val ctx = Dispatchers.Main + exceptionHandler + SupervisorJob()
val scope = CoroutineScope(ctx)
```

------------------------------------------------------------------

## [Part 4.2 : Launching coroutine vs suspending functions](Launching%20coroutine%20vs%20suspending%20functions.md)

코루틴에서 다수의 작업을 동시에 처리하는 방법에는 일반 함수와 일시 중지 함수 2가지 방법이 있습니다.

일반 함수는 코루틴 시작 시 외부 스코프 객체가 필요하며, 코루틴의 완료를 기다리지 않습니다.  
또한 코루틴의 예외는 외부 스코프에서 처리되며, 대부분 전송되고 무시됩니다.  
이렇게 시작된 코루틴은 외부 스코프 객체로부터 컨텔스트를 상속받기에 코루틴을 취소하려면 외부 스코프 자체를 취소해야 합니다.

일시 중지 함수는 모든 코루틴이 완료될 때까지 종료되지 않습니다.  
또한 부모 코루틴을 취소하지 않고도 예외를 던지거나 무시할 수 있어 독립적이며, 예외 처리에 있어서도 더욱 유연합니다.

------------------------------------------------------------------

## [Part 4.3 : Best practices](Best%20practices.md)

- 스코프가 필요한 경우 `async { ... }.await()` 대신 `coroutineScope { ... }`를 사용
- 컨텍스트 변경이 필요한 경우 `withContext`를 사용
- 여러 개의 `async` 작업 시 가독성을 위해 모든 작업에 `async`를 적용
- `withContext(EmptyCoroutineContext)` 대신 `coroutineScope`를 사용
- 여러 비동기 작업 시 `awaitAll` 사용 시 작업에 예외가 발생하면 불필요한 대기 시간을 줄임
- 일시 중지 함수 호출 시 `Disaptchers.Main`을 자주 사용하기에 어떤 스레드에서든 안전하게 호출되어야 함
- 컨텍스트 변경 시 `withContext`를 사용하며, 스레드 블로킹이 예상되는 경우 `IO`, CPU 집약 작업의 경우 `Default`, `Flow`의 경우 `flowOn` 사용
- `Dispatchers.Main.immediate`는 `Dispatchers.Main`의 최적화 버전으로, 코루틴 재배치가 필요하지 않는 경우 회피
- CPU 집약적인 작업을 하는 경우에 `yield()` 사용 시 하나의 코루틴이 시스템 리소스 독점을 방지할 수 있음 
- 코루틴은 부모-자식 관계를 형성하며, 부모 코루틴은 자식 코루틴이 완료될 때까지 완료되지 않음
- `Job`은 상속되지 않는 컨텍스트로, 새로운 코루틴 생성 시 부모 코루틴의 `Job`을 상속이 아닌, 부모 `Job`을 그대로 사용
- 자식 코루틴 예외를 캐치할 때, `withContext`에 `SupervisorJob`을 사용하면 아무런 효과가 없고, `supervisorScope`를 사용하는 것이 효과적
- `withContext`를 통해 `Job`을 명시적으로 설정하면, 구조화된 동시성을 깨드리게 되어 코루틴의 취소가 이루어지지 않을 수 있기에 주의
- `CoroutineScope` 생성 시 여러 코루틴을 독립적으로 작동해야 하는 경우 `SupervisorJob`을 사용해야 함
- 코루틴 스코프는 한 번 취소되면 재사용이 불가능하기에 스코프를 활성화된 상태로 남기고 하위 작업을 취소하려면 `cancelChildren`을 사용
- 안드로이드에서 자동으로 스코프를 관리하려면 `viewModelScope` 혹은 `lifecycleScope` 사용이 편함
- `GlobalScope`는 코루틴을 간단하게 시작할 수 있지만, 자동 취소와 오버라이딩이 어렵기에 사용을 지양
- 코루틴 스코프를 구성할 때를 제외하고는 `Job` 빌더를 사용하지 않는 것이 좋음, 만약 `Job()`을 이용하여 코루틴 생성 시 `Job` 객체를 수동으로 관리해야 함
- `Flow`를 반환하는 함수는 일시 중지 함수가 아니어야 함
- [지연된 단일 값 사용 시 일시 중지 함수가 적절하고, 시간에 따라 여러 값을 사용하는 경우 `Flow`가 적절](../Flow와%20suspend%20functions%20사용시기.md)