# Testing Kotlin Coroutines

일반 함수와 마찬가지로 코루틴을 사용한 함수도 `Fake Data` 또는 `mocks`를 사용하여 특별한 설정이나 복잡한 프로세스 없이 테스트를 수행할 수 있습니다.

```kotlin
class FetchUserUseCase(
    private val repo: UserDataRepository
) {
    suspend fun fetchUserData(): User = coroutineScope {
        val name = async { repo.getName() }
        val friends = async { repo.getFriends() }
        val profile = async { repo.getProfile() }

        User(
            name = name.await(),
            friends = friends.await(),
            profile = profile.await()
        )
    }
}

class FetchUserDataTest {
    @Test
    fun `should construc user`() = runBlocking {
        // given
        val repo = FakeUserDataRepository()
        val useCase = FetchUserUseCase(repo)

        // when
        val result = useCase.fetchUserData()

        // then
        val exceptedUser = User(
            name = "Ben",
            friends = listOf(Friend("some-friend-id-1")),
            profile = Profile("Example description")
        )

        assertEquals(exceptedUser, result)
    }
}
    
class FakeUserDataRepository: UserDataRepository {
    override suspend fun getName(): String = "Ben"
    override suspend fun getFriends(): List<Friend> = listOf(Friend("some-friend-id-1"))
    override suspend fun getProfile(): Profile = Profile("Example description")
}
```

이처럼 대부분 코루틴 테스트 시 `runBlocking`과 `assertion`을 사용하여 테스트를 수행합니다.

---

## Testing time dependencies

테스트 시 함수 실행 시간에 서로 차이가 발생하는 로직이 있는 경우가 있을 수 있습니다.

```kotlin
suspend fun produceCurrentUserSeq(): User {
    val profile = repo.getProfile()
    val friends = repo.getFriends()
    return User(profile, friends)
}

suspend fun produceCurrentUserSym(): User = coroutineScope {
    val profile = async { repo.getProfile() }
    val friends = async { repo.getFriends() }
    User(profile.await(), friends.await())
}
```

위의 두 함수는 동일한 결과를 반환하지만, 첫 번째 함수는 순차적으로 실행되고, 두 번째 함수는 동시에 실행됩니다.
만약 `getProfile`과 `getFriends` 함수가 1초씩 걸린다면 첫 번째 함수는 2s가 걸리고 두 번째 함수는 1s가 걸릴겁니다.

위와 같이 시간적 차이를 테스트하려면 테스트 시 실제 함수 대신 fake 함수와 `delay`를 같이 사용하여 데이터 로딩 시나리오를 시뮬레이션 할 수 있습니다.

```kotlin
class FakeDelayedUserDataRepository : UserDataRepository {
    override suspend fun getProfile(): Profile {
        delay(1000)
        return Profile("Example description")
    }

    override suspend fun getFriends(): List<Friend> {
        delay(1000)
        return listOf(Friend("some-friend-id-1"))
    }
}
```

결과적으로, `produceCurrentUserSeq` 함수는 2s가 걸리고 `produceCurrentUserSym` 함수는 1s가 걸리는 실행 시간에 차이가 있습니다.
그러나 단일 단위 테스트에서 이렇게 많은 시간을 소비하는 것은 바람직하지 않습니다.

대부분의 프로젝트에는 수 천개의 단위 테스트가 있으며 가능한 빨리 모든 테스트를 수행하려고 합니다.

이에 `kotlinx.coroutines.test` 라이브러리는 이러한 문제를 해결하는데 도움을 주며, `StandardTestDispatcher`가 포함되어 있어 테스트 중 코루틴의 가상 시간을 조작할 수 있습니다.
이를 사용하면 `delay`나 다른 시간에 의존하는 함수들도 실제 시간이 아닌 가상의 시간을 기반으로 테스트할 수 있습니다. 

따라서 실제로 오래 걸리는 작업도 테스트에서 거의 즉시 완료될 수 있습니다.

---------------------------------------------------------------

## TestCoroutineScheduler and StandardTestDispatcher

`TestCoroutineScheduler`와 `StandardTestDispatcher`를 사용하면 테스트 중인 코루틴의 `delay` 동작을 조작할 수 있습니다.  
이들은 실제 시간이 아닌 가상 시간을 사용하여 `delay`를 처리하므로 테스트의 전체 실행 시간을 크게 단축시킬 수 있습니다.

```kotlin
fun main() {
    val scheduler = TestCoroutineScheduler()
    
    println(scheduler.currentTime) // 0
    scheduler.advanceTimeBy(1000)
    println(scheduler.currentTime) // 1000
    scheduler.advanceTimeBy(1000)
    println(scheduler.currentTime) // 2000
}
```

`TestCoroutineScheduler`를 코루틴에서 사용하려면 이를 지원하는 디스패처가 필요합니다.  
표준 옵션으로는 `StandardTestDispatcher`를 사용합니다.

`StandardTestDispatcher`는 코루틴이 어느 스레드에서 실행될 것인지만 결정하는 것 이상의 역할을 합니다.
`StandardTestDispatcher`로 시작된 코루틴은 가상 시간을 진행시킬 때까지 실행되지 않습니다.

가장 일반적인 방법으로 이를 수행하는 것은 `advanceUntilIdle`을 사용하는 것으로, 가상 시간을 진행시키고 해당 시간 동안 실제 시간이라면 호출되었을 모든 작업을 호출합니다.

```kotlin
fun main() {
    val scheduler = TestCoroutineScheduler()
    val testDispatcher = StandardTestDispatcher(scheduler)
    
    CoroutineScope(testDispatcher).launch {
        println("Some work 1")
        delay(1000)
        println("Some work 2")
        delay(1000)
        println("Coroutine finished")
    }
    
    println("[${scheduler.currentTime}] Before")
    scheduler.advanceUntilIdle()
    println("[${scheduler.currentTime}] After")
}
// [0] Before
// Some work 1
// Some work 2
// Coroutine finished
// [2000] After
```

`StandardTestDispatcher`는 기본적으로 `TestCoroutineScheduler`를 생성하므로 명시적으로 생성할 필요가 없습니다. 

```kotlin
fun main() {
    val dispatcher = StandardTestDispatcher()
    
    CoroutineScope(dispatcher).launch {
        println("Some work 1")
        delay(1000)
        println("Some work 2")
        delay(1000)
        println("Coroutine finished")
    }
    
    println("[${dispatcher.scheduler.currentTime}] Before")
    dispatcher.scheduler.advanceUntilIdle()
    println("[${dispatcher.scheduler.currentTime}] After")
}

// [0] Before
// Some work 1
// Some work 2
// Coroutine finished
// [2000] After
```

`StandardTestDispatcher`는 시간을 자동으로 진행시키지 않습니다.  
아래 예제와 같이 `delay` 후 코루틴이 재개되지 않고, 무한히 대기 상태에 머무르게 됩니다.

이처럼 코루틴을 자동으로 재개하지 않기에 테스트 중 시간을 수동으로 관리해야 합니다.

```kotlin
fun main() {
    val testDispatcher = StandardTestDispatcher()
    
    runBlocking(testDispatcher) {
        delay(1)
        println("Coroutine finished")
    }
}
// (code runs forever)
```

시간을 진행시키는 또 다른 방법으로는 `advanceTimeBy`를 사용하는 것입니다.

`advanceTimeBy`는 테스트 디스패처의 시간을 인자로 보낸 'ms' 값만큼 진행시킵니다.  
예를 들어 `advanceTimeBy(1)` 호출 시 1ms가 지나고, 이 시간 동안 `delay`로 지연되었던 코루틴들을 재개시킵니다.

그러나 `advanceTimeBy` 사용 시 지정한 시간 '이하'의 지연이 발생된 코루틴만 재개되는 점을 주의해야 합니다.  
만약 2ms 후 예정된 작업은 `advanceTimeBy(1)` 만으로 재개되지 않습니다. 

이러한 경우 `runCurrent`를 호출하여 현재 시점에 예정된 작업들을 명시적으로 실행해야 합니다.

```kotlin
fun main() {
    val testDispatcher = StandardTestDispatcher()
    
    Coroutinescope(testDispatcher).launch {
        delay(1)
        println("Done 1")
    }
    
    Coroutinescope(testDispatcher).launch {
        delay(2)
        println("Done 2")
    }
    
    testDispatcher.scheduler.advanceTimeBy(2) // Done 1
    testDispatcher.scheduler.runCurrent() // Done 2
}

fun main2() {
    val testDispatcher = StandardTestDispatcher()

    CoroutineScope(testDispatcher).launch {
        delay(2)
        print("Done")
    }
    CoroutineScope(testDispatcher).launch {
        delay(4)
        print("Done2")
    }
    CoroutineScope(testDispatcher).launch {
        delay(6)
        print("Done3")
    }
    
    for (i in 1..5) {
        print("~")
        testDispatcher.scheduler.advanceTimeBy(1)
        testDispatcher.scheduler.runCurrent()
    }
}
// ~~Done~~Done2~
```

가상 시간은 실제 시간과 완전히 독립적입니다.

아래 예제를 보면 `Thread.sleep`를 사용하여 스레드를 일시 정지 시켜도 `StandardTestDispatcher`를 사용하는 코루틴은 영향 받지 않습니다.
이는 가상 시간이 실제 시간에 의존하지 않기 때문입니다.

또한 `advanceUntilIdle` 호출은 단 몇 밀리초만 걸리므로 실제 시간을 기다리지 않습니다.

```kotlin
fun main() {
    val dispatcher = StandardTestDispatcher()
    
    CoroutineScope(dispatcher).launch {
        delay(1000)
        println("Coroutine finished")
    }
    
    Thread.sleep(Random.nextLong(2000))
    // Dose not matter how much time we wait here, it will not influence the result
    
    val time = measureTimeMillis { 
        println("[${dispatcher.scheduler.currentTime}] Before")
        dispatcher.scheduler.advanceUntilIdle()
        println("[${dispatcher.scheduler.currentTime}] After")
    }
    
    println("Took $time ms")
}
// [0] Before
// Coroutine finished
// [1000] After
// Took 15 ms
```

위 예제들은 `StandardTestDispatcher`를 사용하고 이를 Scope로 감싸고 있었습니다.
이를 대신해서 `TestScope`를 사용할 수 있으며, 이는 동일한 작업을 수행합니다.

`TestScope`는 추가적인 유용한 메서드와 속성을 제공합니다. 
가장 큰 장점 중 하나는 `advanceUntilIdle`, `advanceTimeBy`, `currentTime` 등의 함수를 사용할 수 있다는 것입니다.

이러한 기능들은 `TestScope`가 내부적으로 사용하는 스케쥴러에 위임되므로, 더 간결하고 직관적인 테스트 코드를 작성할 수 있습니다.

```kotlin
fun main() {
    val scope = TestScope()

    scope.launch {
        delay(1000)
        println("First done")
        delay(1000)
        println("Coroutine finished")
    }

    println("[${scope.currentTime}] Before") // [0] Before
    scope.advanceTimeBy(1000)
    scope.runCurrent() // First done
    println("[${scope.currentTime}] Middle") // [1000] Middle
    scope.advanceUntilIdle() // Coroutine finished
    println("[${scope.currentTime}] After") // [2000] After
}
```

`StandardTestDispatcher`는 Android 개발 환경 중 ViewModel, Presenter, Fragment 등 컴포넌트를 테스트할 때 효과적으로 사용될 수 있습니다.
그러나, 복잡한 코루틴 로직을 테스트할 때에는 `runTest`를 사용하는 것이 더 효과적입니다.

---

## runTest

`runTest` 함수는 코루틴 단위 테스트를 간편하게 해주는 함수로써, 이 함수 사용 시 코루틴 로직을 테스트할 때 실제로 시간을 기다리지 않고 코루틴의 로직을 빠르게 테스트할 수 있습니다.

이는 `TestScope`와 함께 제공되는 가상 시간 관리 기능 덕분에 가능합니다. 
테스트 중 `currentTime`을 체크함으로써 특정 코루틴 로직이 얼마나 시간이 걸리는지, 언제 특정 작업이 실행되는지 확인할 수 있습니다.

```kotlin
class SimpleTest {
    
    @Test
    fun test1() = runTest {
        assertEquals(0, currentTime)
        delay(1000)
        assertEquals(1000, currentTime)
    }
    
    @Test
    fun test2() = runTest {
        assertEquals(0, currentTime)
        coroutineScope {
            launch { delay(1000) }
            launch { delay(1500) }
            launch { delay(2000) }
        }
        assertEquals(2000, currentTime)
    }
}
```

아래 예제는 `runTest`의 강력함을 보여줍니다. 

만약 실제 환경에서 이러한 테스트를 수행한다면, `fetchUserData()`에서 각 함수 호출마다 실제로 1초씩 기다려야할 것입니다.
그러나 `runTest` 사용 시 실제로 시간을 기다리지 않고도 시간에 따른 로직의 실행 순서나 시가능ㄹ 정확하게 확인할 수 있습니다.

```kotlin
@Test
fun `should produce user sequentially`() = runTest {
    // given
    val repo = FakeDelayedUserDataRepository()
    val useCase = ProduceUserUseCase(repo)
    
    // when
    useCase.produceCurrentUserSeq()
    
    // then
    assertEquals(2000, currentTime)
}

@Test
fun `should produce user simultaneously`() = runTest {
    // given
    val repo = FakeDelayedUserDataRepository()
    val useCase = ProduceUserUseCase(repo)
    
    // when
    useCase.produceCurrentUserSym()
    
    // then
    assertEquals(1000, currentTime)
}

class FetchUserUseCase(
    private val repo: UserDataRepository
) {
    suspend fun fetchUserData(): User = coroutineScope {
        val name = async { repo.getName() }
        val friends = async { repo.getFriends() }
        val profile = async { repo.getProfile() }
        
        User(
            name = name.await(),
            friends = friends.await(),
            profile = profile.await()
        )
    }
}

class FetchUserDataTest {
    
    @Test
    fun `should load data concurrently`() = runTest {
        // given
        val repo = FakeDelayedUserDataRepository()
        val useCase = FetchUserUseCase(repo)
        
        // when
        useCase.fetchUserData()
        
        // then
        assertEquals(1000, currentTime)
    }
    
    @Test
    fun `should construct user`() = runTest {
        // given
        val repo = FakeUserDataRepository()
        val useCase = FetchUserUseCase(repo)
        
        // when
        val result = useCase.fetchUserData()
        
        // then
        val exceptedUser = User(
            name = "Ben",
            friends = listOf(Friend("some-friend-id-1")),
            profile = Profile("Example description")
        )
        
        assertEquals(exceptedUser, result)
    }
}

class FakeUserDataRepository: UserDataRepository {
    override suspend fun getName(): String {
        delay(1000)
        return "Ben"
    } 
    override suspend fun getFriends(): List<Friend> {
        delay(1000)
        return listOf(Friend("some-friend-id-1"))
    }
    override suspend fun getProfile(): Profile {
        delay(1000)
        return Profile("Example description")
    }
}

interface UserDataRepository {
    suspend fun getName(): String
    suspend fun getFriends(): List<Friend>
    suspend fun getProfile(): Profile
}

data class User(
    val name: String,
    val friends: List<Friend>,
    val profile: Profile
)

data class Friend(val id: String)
data class Profile(val description: String)
```

---------------------------------------------------------------

## Background scope

`runTest`는 테스트 내에서 코루틴을 실행하기 위한 스코프를 제공합니다.  
이 스코프에서 시작된 모든 코루틴 작업들은 해당 스코프의 생명주기에 종속되며, `runTest` 스코프 내에서 시작된 코루틴 작업이 완료되지 않으면 `runTest`는 반환되지 않습니다.

이는 단위 테스트 시 주의해야 할 부분으로 무한 루프나 끝나지 않는 프로세스 시작 시 테스트가 끝나지 않는 문제가 발생할 수 있습니다.

```kotlin
@Test
fun `should increment counter`() = runTest {
    var i = 0
    launch { 
        while(true) {
            delay(1000)
            i++
        }
    }
    
    delay(1001)
    assertEquals(1, i)
    delay(1000)
    assertEquals(2, i)    
}
```

위와 같은 상황을 위해서 `runTest`는 `backgroundScope`를 제공합니다.

`backgroundScope`는 장기 실행 작업이나, 테스트의 주요 흐름에 직접적인 영향을 주지않는 별도의 프로세스를 시작할 때 유용합니다.  
이렇기에 `runTest`는 `backgroundScope` 작업이 완료될 때까지 기다리지 않고 테스트를 종료합니다.

```kotlin
@Test
fun `should increment counter`() = runTest {
    var i = 0
    backgroundScope.launch {
        while(true) {
            delay(1000)
            i++
        }
    }

    delay(1001)
    assertEquals(1, i)
    delay(1000)
    assertEquals(2, i)
    }
```

---------------------------------------------------------------

## Testing cancellation and context passing

구조화된 동시성은 코루틴을 안전하게 관리하고 예상치 못한 동시성 문제를 방지하기 위한 코루틴의 핵심 원칙 중 하나 입니다.

이 때문에 함수가 구조화된 동시성을 준수하는지 테스트하는 것은 중요합니다.

예를 들어 suspending 함수 내에서 현재 코루틴 컨텍스트를 캡처하면, 해당 컨텍스트가 올바른 값과 상태를 가지고 있는지 테스트를 통해 확인할 수 있습니다.

```kotlin
suspend fun <T, R> Iterable<T>.mapAsync(
    transform: suspend (T) -> R
): List<R> = coroutineScope {
    this@mapAsync.map { 
        async { transform(it) } 
    }.awaitAll()
}
```

`mapAsync`는 원소들을 비동기적으로 매핑하면서 그 순서를 유지해야 합니다.  
이러한 동작은 다음의 테스트로 검증할 수 있습니다.

```kotlin
@Test
fun `should map async and keep elements order`() = runTest {
    val transforms = listOf(
        suspend { delay(3000); "A" },
        suspend { delay(2000); "B" },
        suspend { delay(4000); "C" },
        suspend { delay(1000); "D" }
    )
    
    val res = transforms.mapAsync { it() }
    assertEquals(listOf("A", "B", "C", "D"), res)
    assertEquals(4000, currentTime)
}
```

올바르게 구현된 suspending 함수는 구조화된 병렬성을 존중해야 합니다.

이를 확인하는 가장 쉬운 방법은 부모 코루틴에 `CoroutineName` 컨텍스트를 지정한 다음, 위 `mapAsync`와 같은 변환 함수 내에서 `CoroutineName`이 동일한지 확인하는 것이 있습니다.

코루틴의 현재 컨텍스트를 얻기 위해서는 다음 2가지 방법이 있습니다.

1. suspending 함수의 컨텍스트를 캡처하려면, `currentCoroutineContext` 혹은 `coroutineContext` 프로퍼티를 사용할 수 있습니다.
2. 코루틴 빌더나 스코프 함수 내에서 중첩된 람다에서는, `CoroutineScope`의 `coroutineContext` 프로퍼티가 현재 코루틴 컨텍스트를 제공하는 프로퍼티보다 우선순위가 있기에 `currentCoroutineContext` 함수를 사용해야 합니다.

```kotlin
@Test
fun `should support context propagation`() = runTest {
    val ctx: CoroutineContext? = null
        
    val name1 = CoroutineName("name1")
    withContext(name1) {
        listOf("A").mapAsync {
            ctx = currentCoroutineContext()
            it
        }
        assertEquals(name1, ctx?.get(CoroutineName))
    }
    
    val name2 = CoroutineName("name2")
    withContext(name2) {
        listOf(1, 2, 3).mapAsync {
            ctx = currentCoroutineContext()
            it
        }
        assertEquals(name2, ctx?.get(CoroutineName))
    }
}
```

코루틴의 취소를 테스트하는 가장 쉬운 방법은 내부 함수의 `Job`을 캡처하고 부모 코루틴 취소 후 자식 코루틴 취소를 검증하는 것 입니다.

아래 예제는 `mapAsync` 내의 코루틴이 부모 코루틴과 함께 취소되는지 확인하여 구조화된 병렬성이 잘 동작하는지 검증하는 예시 입니다.

```kotlin
@Test
fun `should support cancellation`() = runTest {
    var job: Job? = null
    
    val parentJob = launch {
        listOf("A").mapAsync {
            job = currentCoroutineContext().job
            delay(Long.MAX_VALUE)
        }
    }
    
    delay(1000)
    parentJob.cancel()
    assertEquals(true, job?.isCancelled)
}
```

---------------------------------------------------------------

## UnconfinedTestDispatcher

`UnconfinedTestDispatcher`와 `StandardTestDispatcher`는 코루틴 테스트를 위한 디스패처로 각각의 디스패처는 코루틴의 실행 방식에 영향을 줍니다.

### StandardTestDispatcher

특정한 스케줄링 동작이 일어나기 전까지는 아무런 코루틴 작업도 실행되지 않습니다.

### UnconfinedTestDispatcher

코루틴이 시작되자마자 첫 번째 `delay`가 호출되기 전까지의 모든 작업들을 즉시 실행합니다. 
만약, 코루틴 내부에서 일련의 작업들을 수행하고 그 중간에 `delay`가 발생되면, 이전의 모든 작업은 즉시 실행되어 그 결과를 출력합니다.

```kotlin
fun main() {
    CoroutineScope(StandardTestDispatcher()).launch {
        println("A")
        delay(1000)
        println("B")
    }
    
    CoroutineScope(UnconfinedTestDispatcher()).launch {
        println("C")
        delay(1000)
        println("D")
    }
}
// C
```

`runBlockingTest` 동작 방식은 `UnconfinedTestDispatcher`를 사용하는 `runTest`와 유사합니다.  
이전에 `runBlockingTest`를 사용하던 테스트 코드를 `runTest`로 마이그레이션을 진행할 때, `UnconfinedTestDispatcher`를 사용하면 더 쉽게 마이그레이션을 진행할 수 있습니다.

```kotlin
@Test
fun testName() = runTest(UnconfinedTestDispatcher()) {
    // ...
}
```

---------------------------------------------------------------

## Using mocks

모킹(mocking)은 테스트에서 외부 시스템이나 복잡한 객체를 대체하기 위해 사용하는 기법입니다.
모킹 라이브러리 사용 시 실제 객체 대신 mock 객체를 생성하여 테스트에서 원하는 동작을 정의할 수 있습니다.

그러나, 모킹에는 다음과 같은 단점이 있습니다.   
인터페이스나 클래스의 구조가 변경될 경우, 모든 mock 객체의 정의도 함께 변경해야 할 수 있습니다.

반면에 `fake`는 실제 객체와 유사한 동작을 수행하는 간단한 객체로 이런 상황에서는 더 유연하고 쉽게 대처할 수 있습니다.

```kotlin
    @Test
    fun `should load data concurrently`() = runTest {
        // given
        val repo = mockk<UserDataRepository>()
        coEvery { repo.getName() } coAnswers { delay(1000); "Ben" }
        coEvery { repo.getFriends() } coAnswers { delay(1000); listOf(Friend("some-friend-id-1")) }
        coEvery { repo.getProfile() } coAnswers { delay(1000); Profile("Example description") }
        
        val useCase = FetchUserUseCase(repo) 
        
        // when
        useCase.fetchUserData()
        
        // then
        assertEquals(1000, currentTime)
    }
```