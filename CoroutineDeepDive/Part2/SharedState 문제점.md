# The problem with shared state

아래의 `UserDownloader` 클래스를 살펴보면, 다음 2가지를 수행할 수 있음을 알 수 있습니다.

- `id`를 통해 사용자 가져오기
- 다운로드된 모든 사용자 가져오기

```kotlin
class UserDownloader(
    private val api: NetworkService
) {
    private val users = mutableListOf<User>()

    fun downloaded(): List<User> = users.toList()

    suspend fun fetchUser(id: Int) {
        val newUser = api.fetchUser(id)
        users.add(newUser)
    }
}
```

> `toList`는 원래의 `mutableList`와는 독립적인 새로운 리스트를 만들어 반환합니다.  
> 이렇게 하면 `downloaded()`를 통해 반환된 리스트가 변경되더라도 원래의 `users` 리스트에는 영향을 주지 않습니다.  
> 이를 'DefensiveCopy'라고 합니다.
>
> 읽기 전용 리스트(`List<User>`)와 읽기-쓰기가 가능한 프로퍼티(`var`)를 사용하면 DefensiveCopy가 필요 없습니다.  
> 그러나 이 경우 새로운 요소를 추가할 때마다 전체 리스트를 새로 만들어야 하므로 성능에 부담이 될 수 있습니다.
>
> 위에서 1번째 방법은 성능은 높지만 데이터 안정성을 보장할 수 없고, 2번째 방법은 데이터의 안정성은 높지만 성능이 저하될 수 있습니다.

위 `UserDownloader` 구현은 다음과 같은 동시성 문제(Concurrency issues)가 발생될 수 있습니다.

1. `users`는 여러 스레드에서 공유되고 있는 상태로 동시에 여러 스레드에서 이 변수를 수정하게 되면 데이터 일관성이 깨질 수 있습니다.
2. `fetchUser()`가 동시에 2개 이상의 스레드에서 실행되면, 각 스레드가 동시에 `users` 리스트를 수정하려고 시도할 것이고, 이러한 접근은 데이터가 충돌 될 가능성을 높입니다.

이 때문에 공유 상태(shared state)인 `users`는 동시 수정(concurrent modifications)이 발생될 수 있기에 적절한 동기화 메커니즘을 사용하여 보호되어야 합니다.

```kotlin
class FakeNetworkService : NetworkService {
    override suspend fun fetchUser(id: Int): User {
        delay(2)
        return User("User$id")
    }
}

suspend fun main() {
    val downloader = UserDownloader(FakeNetworkService())
    coroutineScope {
        repeat(1_000_000) {
            launch { downloader.fetchUser(it) }
        }
    }
    print(downloader.donwloaded().size) // ~998242
}
```

위 코드를 실행한 결과를 보면 여러 스레드가 동일한 인스턴스와 상호 작용하기에 1M 숫자 보다 작은 값을 출력하거나 예외를 발생시킬 수 있습니다.

> Exception in thread "main"  
> java.lang.ArrayIndexOutOfBoundsException: 22  
> at java.util.ArrayList.add(ArrayList.java:463)  
> ...

공유 상태 수정과 관련된 전형적인 문제를 더 명확하게 이해하기 위해 다시 한 번 다른 예제를 살펴보겠습니다.

여러 스레드가 `counter`를 증가 시키며 1000개의 코루틴에서 연산을 1000번 호출하여 원하는 값의 출력은 1M 이어야 합니다.
그러나 동기화 없이 실제 결과는 충돌 때문에 더 적을 것입니다.

```kotlin
var counter = 0

fun main() = runBlocking {
    massiveRun { counter++ }
    println(counter) // 567231 
}

suspend fun massiveRun(
    action: suspend () -> Unit
) = withContext(Dispatchers.Default) {
    repeat(1000) {
        launch {
            repeat(1000) {
                action()
            }
        }
    }
}
```

-------------------------

## Blocking synchronization

공유 상태 수정의 문제를 해결하기 위해서 Java에서는 `synchronized` 블록이나 동기화된 컬렉션과 같은 방법으로 해결할 수 있습니다.

```kotlin
var counter = 0
fun main() = runBlocking {
    val lock = Any()
    massiveRun {
        synchronized(lock) { // We are blocking threads
            counter++
        }
    }

    println(counter) // 1000000
}
```

`synchronized`는 동시성 문제를 해결하지만, 다음과 같은 문제점이 있습니다.

- `synchronized` 블록은 스레드를 차단합니다. 이로 인해 리소스가 낭비되며, 메인 스레드나 제한된 스레드 풀에서는 큰 문제가 발생될 수 잇습니다.
- `synchronized` 블록 내에서는 코루틴의 suspending 함수를 사용할 수 없습니다.

-------------------------

## Atomics

`Atomics`은 간단한 경우에 동시성 문제를 해결하는 또 다른 Java의 솔루션 중 하나 입니다.

`atomic` 연산은 동시에 여러 스레드가 동일한 데이터에 접근할 때 연산이 중간에 방해를 받지 않고 한번에 완료되므로 다른 스레드가 동시에 값을 변경할 수 없습니다.
또한 잠금 메커니즘이 없이 저수준에서 구현되므로 성능에 큰 이점을 가질 수 있습니다.

Java에서는 `atomic values`를 지원하며 여기에서 `AtomicInteger`를 사용하여 동시성 문제가 있는 예제를 수정 해보겠습니다.

```kotlin
private var counter = AtomicInteger()

fun main() = runBlocking {
    massiveRun {
        counter.incrementAndGet()
    }
    print(counter.get()) // 1000000
}
```

위 구현과 같이 `atomic`은 값의 증가와 감소 같은 간단한 연산에서는 유용하지만,
여러 연산을 조합해야 하는 로직에서는 유용하지 않을 수 있습니다.

```kotlin
private var counter = AotomicInteger()
fun main() = runBlocking {
    massiveRun {
        counter.set(counter.get() + 1)
    }
    print(counter.get()) // ~430467
}
```

`UserDownloader`를 안전하게 만들기 위해, `List<*>`를 감싸는 `AtomicReference`를 사용할 수 있으며,
동시성 문제 없이 `users`를 업데이트하기 위해 `getAndUpdate()`을 사용할 수 있습니다.

> `AtomicReference` : Atomic 연산을 제공하는 일반 객체에 대한 참조로 객체의 상태를 원자적으로 읽고 쓸 수 있습니다.  
> `getAndUpdate` : 현재 값을 가져와 주어진 함수로 업데이트 하며, 이 과정에서 다른 스레드가 동시에 값을 변경하는 것을 방지해줍니다.

```kotlin
class UserDownloader(
    private val api: NetworkService
) {
    private val users = AtomicReference(listOf<User>())

    fun downloaded(): List<User> = users.get()

    suspend fun fetchUser(id: Int) {
        val newUser = api.fetchUser(id)
        users.getAndUpdate { it + newUser }
    }
}
```

`atomic`은 주로 간단한 변수나 참조를 안전하게 만들기 위한 목적이며, 복잡한 데이터 구조나 여러 변수가 상호작용해야 하는 경우에는 제한적입니다.

따라서 `atomic`의 사용은 특정 상황에서 유용할 수 있지만, 모든 동시성 문제에 대한 일반적인 해결책으로 볼 수 없습니다.

------------------------------------------------------------

## A dispatcher limited to a single thread

디스패처를 단일 스레드로 제한하여 사용하는 것은 공유 상태 수정 문제를 해결하는 가장 쉬운 방법입니다.

```kotlin
val dispatcher = Dispatchers.IO.limitedParallelism(1)
var counter = 0

fun main() = runBlocking {
    massiveRun {
        withContext(dispatcher) {
            counter++
        }
    }
    println(counter) // 1000000
}
```

디스패처에 단일 스레드로 제한하여 사용하는 방법은 두 가지 방식으로 사용될 수 있습니다.

### coarse-garined thread confinement 방식

이 접근법은 전체함수 또는 큰 코드 블록을 단일 스레드 디스패처로 감싸 공유 상태에 대한 충돌을 쉽게 제거합니다.  
그러나 전체 함수의 멀티 스레딩 기능을 잃게 됩니다.

예를 들어 `api.fetchUser(id)` 같은 네트워크 호출이나 CPU 집약적인 연산이 있는 경우, 단일 스레드에서는 순차적으로 실행되기에 전체 성능이 저하됩니다.
즉, 다중 스레드 환경에서 병렬로 처리할 수 있는 방법이 사라지는 것 입니다.

```kotlin
class UserDownloader(
    private val api: NetworkService
) {
    private val dispatcher = Dispatchers.IO.limitedParallelism(1)
    private val users = mutableListOf<User>()

    suspend fun downloaded(): List<User> = withContext(dispatcher) {
        users.toList()
    }

    suspend fun fetchUser(id: Int) = withContext(dispatcher) {
        val newUser = api.fetchUser(id)
        users += newUser
    }
}
```

### fine-grained thread confinement 방식

이 접근법은 상태를 수정하는 구체적인 부분만 단일 스레드로 실행하도록 제한하는 방식입니다.

예를 들어 공유된 리스트 `users`에 추가 또는 삭제 작업을 수행하는 코드 블록만 단일 스레드 디스패처로 감싸서 실행할 수 있습니다.

이 방식의 장점은 불필요하게 전체 함수나 큰 코드 블록을 단일 스레드로 제한하지 않아도 되기에 병럴 처리의 이점을 더 많이 활용할 수 있으며,
CPU 집약적이거나 블로킹 작업이 있는 경우에도 성능 저하를 최소화할 수 있습니다.

단점으로는 이런 감싸는 작업이 더 복잡하고, 따라서 코드 유지 관리가 더 어려워질 수 있습니다.
또한 어떤 부분을 단일 스레드로 제한할지 결정해야 하므로 초기 설계 단계에서 더 많은 시간을 소모할 수 있습니다.

만약 단순한 suspending 함수의 경우 이 방식으로 처리하더라도 성능 향샹을 기대하기 어렵습니다.
왜냐하면 suspending 함수 자체가 비동기적으로 동작하며, 기본적으로 병렬 처리에 최적화되어 있기 때문입니다.

```kotlin
class UserDownloader(
    private val api: NetworkService
) {
    private val dispatcher = Dispatchers.IO.limitedParallelism(1)
    private val users = mutableListOf<User>()

    suspend fun downloaded(): List<User> = withContext(dispatcher) {
        users.toList()
    }

    suspend fun fetchUser(id: Int) {
        val newUser = api.fetchUser(id)
        withContext(dispatcher) { users += newUser }
    }
}

```

이처럼 단일 스레드로 제한된 디스패처를 사용하는 것은 간단하며, 표준 디스패처와 동일한 스레드 풀을 공유하기에 효율적이기도 합니다.

---

## Mutex

`Mutex`는 상호 배제(Mutual Exclusion)의 줄임말로, 여러 코루틴이 동시에 특정 리소스에 접근하는 것을 막기 위해 사용됩니다.
이렇게 하면 리소스가 변경되는 도중에 다른 코루틴이 그 상태를 변경할 수 없으므로, 데이터 일관성을 유지할 수 있습니다.

`lock()`을 호출한 코루틴은 `unlock()`을 호출하기 전까지 그 리소스에 대한 전용 접근 권한을 갖습니다.

이로 인해 다른 코루틴들이 리소스에 접근 시도 시 대기 상태에 놓이게 되며, 이들은 대기열에 쌓이게 됩니다.  
첫 번째 코루틴이 `unlock()`을 호출하면 대기열에서 다음 코루틴이 `lock()`을 통과할 수 있습니다.

이런 방식으로여러 코루틴 사이에서 리소스를 안전하게 공유할 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    repeat(5) {
        launch {
            dealyAndPrint()
        }
    }
}

val mutex = Mutex()
suspend fun delayAndPrint() {
    mutex.lock()
    delay(1000)
    println("Done")
    mutex.unlock()
}

// 1s delay
// Done
// 1s delay
// Done
// 1s delay
// Done
// 1s delay
// Done
// 1s delay
// Done
```

그러나, 두 함수 사이에 발생할 수 있는 예외나 조기 리턴(premature return)으로 인해 `unlock()`이 호출되지 않는 데드락 문제가 발생할 수 있기 때문에 `lock()`과 `unlock()`을
직접 사용하는 것은 위험합니다.

이 대신, `withLock()`을 사용할 수 있습니다.

`withLock()`은 내부적으로 `try-finally` 블록을 사용하여 `lock`을 안전하게 관리합니다.  
코드 블록 내에서 예외가 발생하더라도 `finally` 블록에서 `unlock`이 호출되므로 리소스를 안전하게 해제할 수 있습니다.

```kotlin
val mutex = Mutex()
var counter = 0

fun main() = runBlocking {
    massiveRun {
        mutex.withLock {
            counter++
        }
    }
    println(counter) // 1000000
}
```

`Mutex`의 중요한 점은 스레드를 차단하는 것이 아닌, 코루틴을 일시 중지 시킨다는점 입니다.  
이는 단일 스레드로 병렬성이 제한된 디스패처를 사용하는 것과 비교할 때, `Mutex`는 더 안전하고 경령화되어 있으며 경우에 따라서는 더 나은 성능을 제공할 수 있습니다.

주의할 점으로는 `Mutex` 사용 시 `lock`을 2번 지나려고하면 데드락에 빠질 수 있습니다.

```kotlin
suspend fun main() {
    val mutex = Mutex()
    println("Started")

    mutex.withLock {
        mutex.withLock {
            println("Will never be printed")
        }
    }
}

// Started
// runs forever
```

그 다음 주의할 점으로는 코루틴이 중단되었을 때도 잠금이 해제되지 않습니다.  
이는 일반적으로 코루틴이 일시 중지되면 다른 코루틴이 작업을 계속할 수 있어야 합니다.   
그러나, `Mutex`로 인해 잠겨있으면 해당 리소스에 접근할 수 없으므로, 다른 코루틴도 중단될 수 있습니다.

아래 코드를 보면 `delay` 사용 시 코루틴은 일시 중지되지만, `Mutex`는 잠겨 있는 상태를 유지합니다.  
이것은 코루틴이 다시 재개될 때까지 다른 코루틴이 `Mutex`로 보호되는 리소스에 접근할 수 없음을 의미합니다.

```kotlin
class MessageRepository {
    private val messages = mutableListOf<String>()
    private val mutex = Mutex()

    suspend fun add(message: String) = mutex.withLock {
        delay(1000)
        messages.add(message)
    }
}

suspend fun main() {
    val repo = MessageRepository()
    val timeMillis = measureTimeMillis {
        coroutineScope {
            repeat(5) {
                launch {
                    repo.add("Message$it")
                }
            }
        }
    }

    println(timeMillis) // ~5120
}
```

단일 스레드로 제한된 디스패처를 사용할 때는 위와 같은 문제가 발생되지 않습니다.  
`delay`나 네트워크 호출이 코루틴을 중단시킬 때, 그 스레드는 다른 코루틴에 의해 사용될 수 있습니다.

```kotlin
class MessageRepository {
    private val messages = mutableListOf<String>()
    private val dispatcher = Dispatchers.IO.limitedParallelism(1)

    suspend fun add(message: String) = withContext(dispatcher) {
        delay(1000)
        messages.add(message)
    }
}

suspend fun main() {
    val repo = MessageRepository()
    val timeMillis = measureTimeMillis {
        coroutineScope {
            repeat(5) {
                launch {
                    repo.add("Message$it")
                }
            }
        }
    }

    println(timeMillis) // ~1058
}
```

마지막으로 `Mutex`를 전체 함수에 적용하면, 그 함수가 수행되는 동안 다른 코루틴이 해당 리소스에 접근하게 될 수 없습니다.  
이러한 방식은 세분화되지 않은(coarse-grained) 접근법이라고도 하며, 이는 동시성을 제어하는 데 있어서 비효율적일 수 있습니다.

아래 예시에서는 `Mutex`를 사용하는 것이 아닌, 단일 스레드로 제한된 디스패처를 사용하는 것이 더 효율적일 것입니다.

```kotlin

class MongoUserRepository(
    // ...
) : UserRepository {
    private val mutex = Mutex()

    override suspend fun updateUser(
        userId: String,
        userUpdate: UserUpdate
    ): Unit = mutext.withLock {
        val currentUser = getUser(userId) // Deadlock!
        deleteUser(userId) // Deadlock!
        addUser(currentUser.updated(userUpdate)) // Deadlock!
    }

    override suspend fun getUser(userId: String): User = mutex.withLock {
        // ...
    }

    override suspend fun deleteUser(userId: String): Unit = mutex.withLock {
        // ...
    }

    override suspend fun addUser(user: User): User = mutex.withLock {
        // ...
    }
}
```

---

## Semaphore

- `Mutex` : 단일 잠금만을 제공하여 1번에 하나의 코루틴만 해당 리소스에 접근을 허용합니다. 
- `Semaphore` : 허가 개수에 따라 그 수 만큼의 코루틴이 동시에 리소스에 접근을 허용합니다.

`Mutex`는 `lock`, `unlock`, `withLock`과 같은 함수를 사용하여 잠금과 해제를 관리합니다.
반면 `Semaphore`는 `acquire`, `release`, `withPermit`과 같은 함수를 사용하여 허가를 얻고 반환합니다.

```kotlin
suspend fun main() = coroutineScope {
    val semaphore = Semaphore(2)
    repeat(5) {
        launch {
            semaphore.withPermit {
                delay(1000)
                println("$it ")
            }
        }
    }
}
// 0 1
// 1s delay
// 2 3
// 1s delay
// 4
```

`Semaphore`가 여러 허가를 관리할 수 있다고 하여도, 이것은 공유 상태에 대한 문제를 해결하진 않습니다.  
그러나 동시 요청의 수를 제한하는 데 사용될 수 있으므로 속도 제한을 구현하여 서비스의 안전성을 유지하는 데 도움이 됩니다.

```kotlin
class LimitedNetworkUserRepository(
    private val api: UserApi
) {
    // We limit to 10 concurrent requests
    private val semaphore = Semaphore(10)

    suspend fun requestUser(userId: String) =
        semaphore.withPermit { api.requestUser(userId) }
}
```