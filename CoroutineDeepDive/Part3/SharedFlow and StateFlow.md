# SharedFlow and StateFlow

## SharedFlow

`SharedFlow`는 여러 코루틴이 동시에 구독하고 데이터를 수집할 수 있는 `Flow`의 특수한 형태입니다.  
`MutableSharedFlow`는 `SharedFlow`의 가변성 타입으로 데이터를 발행(`emit`)할 수 있는 기능이 추가되어 있습니다.

이것은 브로드캐스트 채널의 개념과 매우 유사하며, 한 곳에서 메시지를 보내면, 해당 채널에 연결된 모든 수신자 그 메시지를 수신할 수 있습니다.
마찬가지로, `MutableSharedFlow`에서 데이터를 발행하면, 해당 `SharedFlow`를 구독하는 모든 코루틴이 해당 데이터를 수신할 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    val mutableSharedFlow = MutableSharedFlow<String>(replay = 0)
    // or MutableSharedFlow<String>()

    launch {
        mutableSharedFlow.collect {
            println("#1 received $it")
        }
    }
    
    launch {
        mutableSharedFlow.collect {
            println("#2 received $it")
        }
    }
    
    delay(1000)
    mutableSharedFlow.emit("Message1")
    mutableSharedFlow.emit("Message2")
}
// 1s delay
// #1 received Message1
// #2 received Message1
// #1 received Message2
// #2 received Message2
// (program never end)
```

`MutableSharedFlow`는 닫을 수 있는 메커니즘이 없기에, 한 번 시작된 수신 대기 상태의 코루틴은 해당 `SharedFlow`로 부터 메시지가 발행되기를 계속 기다립니다.
따라서 이러한 코루틴들은 자동으로 종료되지 않습니다.

이러한 문제를 해결하기 위해서는 해당 스코프에 속한 모든 코루틴을 명시적으로 취소해야 합니다.

---

`MutaleSharedFlow`는 `replay` 파라미터를 통해 최근에 발행된 값 중 특정 개수만큼의 값을 버퍼에 저장하는 기능을 제공합니다.  
예를 들어, `replay = 2` 설정 시 마지막으로 발행된 2개의 값만 버퍼에 저장되고, 새로운 수신자가 해당 `SharedFlow`를 구독하기 시작할 때, 이 2개의 값이 먼저 전달됩니다.

이러한 기능은 특정 상황에서 유용할 수 있는데, 어떤 상태나 정보를 전달하는 `SharedFlow`에서 최신 상태의 스냅샷을 바로 얻고 싶은 수신자가 있을 떄 유용합니다.

또한, `resetReplayCache`를 통해 `MutableSharedFlow`의 버퍼에 저장된 값들을 초기화할 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    val mutableSharedFlow = MutableSharedFlow<String>(replay = 2)
    
    mutableSharedFlow.emit("Message1")
    mutableSharedFlow.emit("Message2")
    mutableSharedFlow.emit("Message3")
    
    println(mutableSharedFlow.replayCache) // [Message2, Message3]
    
    launch {
        mutableSharedFlow.collect {
            println("#1 received $it")
        }
        // #1 received Message2
        // #1 received Message3
    }
    delay(100)
    mutableSharedFlow.resetReplayCache()
    println(mutableSharedFlow.replayCache) // []
}
```

Kotlin은 명확한 책임 분리를 중요하게 생각하며, 수신만을 위해 시용되는 인터페이스와 수정을 위해 사용되는 인터페이스들 사이에 구별이 이루어집니다.

`SharedFlow`는 데이터를 관찰하고 수신하는 데 사용되며, `Flow`에서 상속받습니다.
`FlowCollector`는 데이터를 발행하는데 사용됩니다.

`MutableSharedFlow`는 `SharedFlow`와 `FlowCollector` 모두의 기능을 결합한 것으로, 데이터를 발행하고 관찰할 수 있는 기능을 갖춘 인터페이스입니다.  
이러한 구별은 코드의 의도를 명확하게 하고, 특정 역할을 위한 인터페이스만을 사용하여 코드의 안전성을 높일 수 있습니다.

```kotlin
interface MutableShardedFlow<T> : SharedFlow<T>, FlowCollector<T> {
    fun tryEmit(value: T): Boolean
    val subscriptionCount: StateFlow<Int>
    fun resetReplayCache()
}

interface SharedFlow<out T>: Flow<T> {
    val replayCache: List<T>
}

interface FlowCollector<in T> {
    suspend fun emit(value: T)
}

suspend fun main() = coroutineScope {
    val mutableSharedFlow = MutableSharedFlow<String>()
    val sharedFlow: SharedFlow<String> = mutableSharedFlow
    val collector: FlowCollector<String> = mutableSharedFlow
    
    launch {
        mutableSharedFlow.collect {
            println("#1 received $it")
        }
    }
    
    launch {
        sharedFlow.collect {
            println("#2 received $it")
        }
    }
    
    delay(1000)
    mutableSharedFlow.emit("Message1")
    collector.emit("Message2")
}

// 1s delay
// #1 received Message1
// #2 received Message1
// #1 received Message2
// #2 received Message2
```

```kotlin
class UserProfileViewModel {
    private val _userChanges = MutableSharedFlow<UserProfile>()
    val userChanges: SharedFlow<UserProfile> = _userChanges
    
    fun onCreate() {
        viewModelScope.launch {
            userChanges.collect(::applyUserChange)
        }
    }
    
    fun onNameChanged(newName: String) {
        _userChanges.emit(NameChange(newName))
    }
    
    fun onPublicKeyChanged(newPublicKey: String) {
        _userChanges.emit(PublicKeyChange(newPublicKey))
    }
}
```

---

### shareIn

`Flow`는 변화를 연속적으로 관찰하고 반응하는 데 사용됩니다.   
때로는 여러 구성요소나 클래스들이 동일한 변화나 이벤트 스트림에 관심을 가질 수 있습니다.  
이러한 경우 하나의 `Flow` 소스를 여러 구독자와 공유하는 필요성이 생깁니다.

`SharedFlow`는 이러한 요구 사항을 충족하기 위해 설계되었으며, 여러 구독자가 동일한 `Flow`를 동시에 구독할 수 있는 Hot Stream 입니다.

여기서 `shareIn`은 일반적인 `Flow`를 `SharedFlow`로 쉽게 변환할 수 있는 메서드입니다.  
이렇게 하면 원래의 `Flow`에 연결된 모든 변화나 이벤트가 여러 수신자로 전달될 수 있습니다.  
즉, `shareIn`을 사용하면 기존의 Cold Stream Flow -> Hot Stream Flow로 변환할 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    val flow = flowOf("A", "B", "C").onEach { delay(1000) }
    
    val sharedFlow: SharedFlow<String> = 
        flow.shareIn(
            scope = this,
            started = SharingStarted.Eagerly,
            // replay = 0 (default)
        )
    
    delay(500)
    launch { sharedFlow.collect { println("#1 $it") } }
    
    delay(1000)
    launch { sharedFlow.collect { println("#2 $it") } }
    
    delay(1000)
    launch { sharedFlow.collect { println("#3 $it") } }
}
// 1s delay
// #1 A
// 1s delay
// #1 B
// #2 B
// 1s delay
// #1 C
// #2 C
// #3 C
```

`shareIn`은 기본 `Flow`를 `SharedFlow`로 변환하는 데 사용됩니다. 이 함수는 아래와 같은 3개의 파라미터를 받습니다.

- `CoroutineScope` : 이 스코프 내에서 `Flow`를 수집하고, `SharedFlow`로 요소를 전송하는 코루틴이 시작됩니다. 이 스코프는 코루틴이 종료될 때 까지 존속됩니다.
- `Started` : 이 파라미터는 `SharedFlow`가 어느 시점에 데이터를 수신하기 시작할 지 결정합니다. 그 시점은 구독 중인 수신자의 수에 따라 다를 수 있습니다.
- `Replay` : `SharedFlow`에 의해 저장되고 새로운 구독자에게 전송될 최근의 값의 수를 나타냅니다.

---

이 중 `started` 파라미터의 여러 옵션은 `SharedFlow`가 데이터를 수신하기 시작하는 시점을 조절하는 데 사용됩니다.  
이것은 여러 수신자들이 `SharedFlow`를 구독하는 다양한 상황에서 유용합니다.

#### SharingStarted.Eagerly

이 옵션은 `shareIn` 함수에 전달될 때 해당 `SharedFlow`가 즉시 데이터를 수신하기 시작하게 됩니다.  
즉, 별도의 수신자가 구독을 시작하기 전에도 원본 `Flow`에서 데이터를 수신하게 됩니다.

만약 `replay = 0`일 경우 새로운 수신자가 구독을 시작하기 전 발행된 모든 데이터는 저장되지 않고 손실되지만, `replay` 값이 0보다 큰 경우 해당 값만큼의 데이터가 저장되어 새로운 수신자가 구독을 시작하면 즉시 전달됩니다.

따라서 `SharingStarted.Eagerly` 옵션 사용 시 `replay` 값을 적절하게 설정하여 원하는 데이터를 손실하지 않도록 주의해야 합니다.

```kotlin
suspend fun main() = coroutineScope {
    val flow = flowOf("A", "B", "C")
    
    val sharedFlow: SharedFlow<String> = flow.shareIn(
        scope = this,
        started = SharingStarted.Eagerly,
    )
    
    delay(100)
    launch { sharedFlow.collect { println("#1 $it") } }
    println("Done")
}
// 0.1s delay
// Done
```

#### SharingStarted.Lazily

이 옵션을 사용하면, `SharedFlow`가 첫 번째 구독자가 등장할 때까지 데이터를 수신하는 것을 지연합니다.  
즉, 처음으로 `SharedFlow`를 구독하는 구독자가 나타날때까지 아무런 데이터도 수신되지 않습니다.

이 설정의 장점은 첫 번째 구독자가 발행된 모든 값을 얻는 것을 보장하며, 그 후의 구독자들은 `replay`에 설정에 따라 최근 데이터만 받을 수 있습니다.  
또한, 모든 구독자가 `SharedFlow`의 구독을 중단하더라도 원본 `Flow`는 계속 활성화되어 데이터를 수신하게 됩니다.  
이 때, `replay` 설정에 따라 가장 최근의 데이터들만 캐시에 저장되며, 이 후 구독자는 이 캐시된 데이터부터 수신을 시작합니다.

```kotlin
suspend fun main() = coroutineScope {
    val flow1 = flowOf("A", "B", "C")
    val flow2 = flowOf("D").onEach { delay(1000) }
    
    val sharedFlow: SharedFlow<String> = 
        merge(flow1, flow2).shareIn(scope = this, started = SharingStarted.Lazily)
    
    delay(100)
    launch { sharedFlow.collect { println("#1 $it") } }
    
    delay(1000)
    launch { sharedFlow.collect { println("#2 $it") } }
}
// 0.1s delay
// #1 A
// #1 B
// #1 C
// 1s delay
// #2 D
// #1 D
```

#### SharingStarted.WhileSubscribed

이 옵션은 `SharedFlow`의 동작을 구독자의 존재에 따라 동적으로 제어할 수 있게 합니다.

1. 최초 구독자가 등장하면 `SharedFlow`는 원본 `Flow`로 부터 데이터 수신을 시작합니다.
2. 마지막 구독자가 사라지면 `SharedFlow`는 데이터 수신을 일시 중지합니다.
3. 데이터 수신을 중지한 상태에서 새로운 구독자가 나타날 때 `SharedFlow`는 다시 데이터 수신을 시작합니다.

또한 2가지 파라미터를 제공하여 더욱 세밀하게 조절할 수 있습니다.

- `stopTimeoutMillis` : 마지막 구독자가 사라진 후, 얼마동안 데이터를 계속 수신할 것인지를 결정합니다. 기본 값은 `0` 입니다.
- `replayExpriationMillis` : `SharedFlow`가 중지된 후 얼마동안 캐시된 데이터를 유지할 것인지를 결정합니다. 기본 값은 `Long.MAX_VALUE` 입니다.

```kotlin
suspend fun main() = coroutineScope {
    val flow = flowOf("A", "B", "C", "D")
        .onStart { println("Started") }
        .onCompletion { println("Finished") }
        .onEach { delay(1000) }
    
    val sharedFlow = flow.shareIn(
        scope = this,
        started = SharingStarted.WhileSubscribed()
    )
    
    delay(3000)
    launch { println("#1 ${sharedFlow.first()}") }
    laucnh { println("#2 ${sharedFlow.take(2).toList()}") }
    
    delay(3000)
    launch { println("#3 ${sharedFlow.first()}") }
}
// 3s delay - Started
// 1s delay - #1 A
// 1s delay - #2 [A, B]
// Finished
// 1s delay - Started
// 1s delay - #3 A
// Finished
```

---

`shareIn`을 사용하여 저장된 `Location`이 시간이 지남에 따라 어떻게 변하는지 관찰해야 한다고 가정해보면, 
아래 예제는 Android-Room 라이브러리를 사용하여 DTO가 어떻게 구현될 수 있는지에 대한 것입니다.

```kotlin
@Dao
interface LocationDao {
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertLocation(location: Location)
    
    @Query("DELETE FROM location_table")
    suspend fun deleteLocations()
    
    @Query("SELECT * FROM location_table ORDER BY time")
    fun observeLocations(): Flow<List<Location>>
}
```

여러 서비스들이 이 `Location`에 의존해야 한다면, 각 서비스가 DB를 따로 관찰하는 것은 최적의 방법이 아니며
`Location`의 변경사항을 수신하여 `SharedFlow`로 공유하는 서비스를 만들 수 있습니다.

이 때 구독자가 즉시 마지막 위치 목록을 받기 원하는 경우 `replay = 1`을 통해 얻을 수 있고,
변경에만 수신하고 싶은 경우 `replay = 0`으로 설정하면 됩니다.

```kotlin
class LocationService(
    locationDao: LocationDao,
    scope: CoroutineScope
) {
    private val locations = locationDao.observeLocations()
        .shareIn(
            scope = scope,
            started = SharingStarted.WhileSubscribed(),
        )
    
    fun observeLocations(): Flow<List<Location>> = locations
}
```