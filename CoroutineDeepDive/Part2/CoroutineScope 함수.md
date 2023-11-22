# Coroutine scope functions

## Approaches that were used before coroutine scope functions were introduced

아래 예제는 CoroutineScope가 도입되기 전 일반적으로 suspension 함수를 호출하여 2개의 엔드포인트에서 데이터를 가져오는 방법입니다.

이 방법의 문제점은 동시성이 없습니다. 즉, 하나의 엔드포인트에서 데이터를 가져오는데 1s가 걸린다면 함수는 2s가 소모됩니다.

```kotlin
// Data loaded sequentially, not simultaneously
suspend fun getUserProfile(): UserProfileData {
    val user = getUserData() // 1s
    val notifications = getNotifications() // 1s

    // 2s
    return UserProfileData(
        user = user,
        notifications = notifications,
    )
}
```

2개의 suspension point를 동시에 실행하는 가장 쉬운 방법은 `async`를 활용하는 것입니다.  
그러나 `async`는 스코프(`Scope`)가 필요하며, `GlobalScope`를 사용하는 것은 좋지 않습니다.

```kotlin
// DON'T DO THAT
suspend fun getUserProfile(): UserProfileData {
    val user = GlobalScope.async { getUserData() }
    val notifications = GlobalScope.async { getNotifications() }

    // 1s
    return UserProfileData(
        user = user.await(),
        notifications = notifications.await(),
    )
}
```

`GlobalScope`는 `EmptyCoroutineContext`와 함께 있는 `CoroutineScope` 입니다.

```kotlin
public object GlobalScope : CoroutineScope {
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}
```

`GlobalScope` 사용 시 다음과 같은 특징을 지닙니다.

- 부모 코루틴이 취소되더라도 `GlobalScope`에서 실행된 `async` 코루틴은 독립적으로 계속 실행됩니다.
- 보통 코루틴은 부모 코루틴의 스코프와 컨텍스트를 상속받습니다. 그러나 `GlobalScope` 사용 시 이러한 상속이 발생되지 않습니다.

위와 같은 특징은 다음과 같은 결과가 발생됩니다.

- 코루틴이 취소되지 않고 계속 실행되면, 불필요한 메모리와 CPU를 사용하게 되어 시스템 성능에 문제를 줄 수 있습니다.
- 상속이 발생되지 않으면 항상 Default Dispatcher에서 실행되며 부모 컨텍스트(`Dispatcher`, `Job` 등)를 고려하지 않아 예기치 못한 동작이 발생될 수 있습니다.
- 일반적으로 UnitTest 도구들도 구조적 동시성 메커니즘을 기반으로 동작하기에 이러한 메커니즘이 없으면 UnitTest가 어려워 집니다.

그렇다고 해서 아래와 같이 `CoroutineScope`를 인자로 전달하는 방법은 좋지 않습니다.

```kotlin
// DON'T DO THAT
suspend fun getUserProfile(scope: CoroutineScope): UserProfileData {
    val user = scope.async { getUserData() }
    val notifications = scope.async { getNotifications() }

    // 1s
    return UserProfileData(
        user = user.await(),
        notifications = notifications.await(),
    )
}

// or

// DON'T DO THAT
suspend fun CoroutineScope.getUserProfile(): UserProfileData {
    val user = async { getUserData() }
    val notifications = async { getNotifications() }

    // 1s
    return UserProfileData(
        user = user.await(),
        notifications = notifications.await(),
    )
}
```

위와 같은 방법의 문제점으로는 다음과 같습니다.

1. 중첩된 함수 호출을 관리하는 경우에 스코프를 여러 함수에 전달할 시 코드와 스코프 관리가 어려워집니다.
2. `SupervisorJob`이 아닌 `Job`을 사용하여 `async`를 실행하다가 블록 내부에서 예외가 발생되면 `CoroutineScope` 내의 모든 작업이 중단됩니다.
3. `CoroutineScope`에 직접 접근하여 `cancel()` 메서드 호출 시 해당 스코프를 어디서든 취소할 수 있습니다.  
   이는 예상치 못한 동작을 초래할 수 있고, 프로그램 안정성 측면에서 좋지 않습니다.

```kotlin
data class Details(val name: String, val followers: Int)
data class Tweet(val text: String)

fun getFollowersNumber(): Int = throw Error("Service exception")

suspend fun getUserName(): String {
    delay(500)
    return "marcinmoskala"
}

suspend fun getTweets(): List<Tweet> {
    return listOf(Tweet("Hello, world"))
}

suspend fun CoroutineScope.getUserDetails(): Details {
    val userName = async { getUserName() }
    val followersNumber = async { getFollowersNumber() }
    return Details(userName.await(), followersNumber.await())
}

fun main() = runBlocking {
    val details =
        try {
            getUserDetails()
        } catch (e: Error) {
            null
        }

    val tweets = async { getTweets() }
    println("User: $details")
    println("Tweets: ${tweets.await()}")
}
// Only Exception...
```

위 코드는 `getUserDetails()`를 가져오는데 실패하더라도 `getTweets()`을 호출하는 코드입니다.
그러나, 예외가 발생하게되면 `async`가 중단되고, 이로 인해 전체 스코프(`runBlocking`)와 프로그램이 종료됩니다.

이러한 이유로, 특정 작업에서 발생하는 에외가 다른 작업에 영향을 미치지 않게 하려면 `coroutineScope`를 사용하는 것이 좋습니다.

---

## coroutineScope

`coroutineScope`는 일시 정지 가능한 코드 블록을 실행하기 위한 새로운 코루틴 스코프를 생성합니다.  
이 스코프 내에서 시작된 코루틴들을 모두 완료될 때까지 `coroutineScope` 함수는 반환되지 않습니다.

```kotlin
suspend fun <R> coroutineScope(
    block: suspend CoroutineScope.() -> R
): R
```

`launch`나 `async`는 새로운 코루틴을 생성하고 즉시 다음 코드 라인으로 넘어가지만
이와 반대로 `coroutineScope`는 해당 블록에서 시작된 모든 코루틴이 완료될 때까지 현재 코루틴을 일시 중단합니다.

이러한 이유로 `coroutineScope`는 직렬 작업을 수행할 때 유용하게 사용될 수 있으며, 코드 복잡성을 줄일 수 있습니다.

```kotlin
fun main() = runBlocking {
    val a = coroutineScope {
        delay(1000)
        10
    }
    println("a is calculated")
    val b = coroutineScope {
        delay(1000)
        20
    }
    println(a) // 10
    println(b) // 20
}
// 1s delay
// a is calculated
// 1s delay
// 10
// 20
```

`coroutineScope`는 상위 스코프의 `coroutineContext`를 상속받아, 자식 코루틴이 해당 컨텍스트에서 실행됩니다.
예를 들어 부모 코루틴이 특정 디스패처를 사용하고 있다면, `coroutineScope` 내의 코루틴도 같은 디스패처를 사용하게 됩니다.

또한 `coroutineScope`는 내부에서 생성된 모든 자식 코루틴이 완료되어야지만, 자기 자신을 종료할 수 있습니다.

```kotlin
suspend fun longTask() = coroutineScope {
    launch {
        delay(1000)
        val name = coroutineContext[CoroutineName]?.name
        println("[$name] Finished task 1")
    }

    launch {
        delay(2000)
        val name = coroutineContext[CoroutineName]?.name
        println("[$name] Finished task 2")
    }
}

fun main() = runBlocking(CoroutineName("Parent")) {
    println("Before")
    longTask()
    println("After")
}

// Before
// 1s delay
// [Parent] Finished task 1
// 1s delay
// [Parent] Finished task 2
// After
```

`coroutineScope`가 속한 부모 코루틴이 취소된다면, `coroutineScope` 내에서 생성된 모든 자식 코루틴들도 취소됩니다.

```kotlin
suspend fun longTask() = coroutineScope {
    launch {
        delay(1000)
        val name = coroutineContext[CoroutineName]?.name
        println("[$name] Finished task 1")
    }

    launch {
        delay(2000)
        val name = coroutineContext[CoroutineName]?.name
        println("[$name] Finished task 2")
    }
}

fun main(): Unit = runBlocking {
    val job = launch(CoroutineName("Parent")) {
        longTask()
    }

    delay(1500)
    job.cancel()
}
// [Parent] Finished task 1
```

`coroutineScope` 또는 그 안의 자식 코루틴에서 예외가 발생되면 해당 스코프에 있는 모든 자식 코루틴들이 취소되고 예외를 다시 던집니다.
이는 부모 코루틴에서 해당 예외를 확인하고 처리할 수 있도록 해줍니다.

```kotlin
data class Details(val name: String, val followers: Int)
data class Tweet(val text: String)

class ApiException(val code: Int, message: String) : Throwable(message)

fun getFollowersNumber(): Int = throw ApiException(500, "Service unavailable")

suspend fun getUserName(): String {
    delay(500)
    return "marcinmoskala"
}

suspend fun getTweets(): List<Tweet> {
    return listOf(Tweet("Hello, world"))
}

suspend fun getUserDetails(): Details = coroutineScope {
    val userName = async { getUserName() }
    val followersNumber = async { getFollowersNumber() }
    Details(userName.await(), followersNumber.await())
}

fun main() = runBlocking<Unit> {
    val details =
        try {
            getUserDetails()
        } catch (e: ApiException) {
            null
        }

    val tweets = async { getTweets() }
    println("User: $details")
    println("Tweets: ${tweets.await()}")
}

// User: null
// Tweets: [Tweet(text=Hello, world)]
```

`coroutineScope`는 여러 호출을 동시에 실행하기에 좋습니다.

```kotlin
suspend fun getUserProfile(): UserProfileData = coroutineScope {
    val user = async { getUserData() }
    val notifications = async { getNotifications() }

    // 1s
    UserProfileData(
        user = user.await(),
        notifications = notifications.await(),
    )
}
```

`runBlocking`대신 suspension 함수의 main body를 감싸는 데에 `coroutineScope`를 사용하는 것이 자주 사용됩니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    launch {
        delay(1000)
        println("World")
    }
    println("Hello, ")
}
// Hello,
// 1s delay
// World
```

아래 두 함수는 순차적으로 호출하는것과 동시에 호출하는 차이가 있습니다.

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

---

## Coroutine scope functions

코루틴 스코프 함수는 suspending 함수 내에서 새로운 코루틴 스코프를 만들거나 기존의 스코프를 변형하는데 사용됩니다.
이는 코루틴의 실행 컨텍스트를 제어하거나, 오류 처리 방식을 정의하거나, 타임아웃을 설정하는 등 다양한 작업을 가능하게 합니다.

`coroutineScope`와 비슷한 동작을 하는 스코프를 생성하는 여러 함수들이 있습니다.

- `supervisorScope`는 `Job` 대신 `SupervisorJob`을 사용합니다.
- `withContext`는 코루틴이 다른 디스패처나 다른 컨텍스트에서 수정되어야 할 때 사용되는 `coroutineScope`입니다.
- `withTimeout`은 코루틴이 정해진 시간에 완료되지 않으면 `TimeoutException`을 발생 시키는 `coroutineScope`입니다.

### 코루틴 스코프 함수와 코루틴 빌더의 차이점

코루틴 스코프 함수와 코루틴 빌더는 종종 혼동되지만, 이 둘을 개념적으로나 실질적으로나 매우 다릅니다.

| 코루틴 스코프 함수                                                        | 코루틴 빌더                                 |
|-------------------------------------------------------------------|----------------------------------------|
| `coroutineScope`, `supervisorScope`, `withContext`, `withTimeout` | `launch`, `async`, `produce`           |
| suspending 함수                                                     | Extension 함수 or `CoroutineScope`       |
| suspending 함수의 `continuation`에서 코루틴 컨텍스트를 가져옵니다                   | `CoroutineScope` 수신자에서 코루틴 컨텍스트를 가져옵니다 |
| 예외는 일반 함수에서처럼 던져집니다                                               | 예외는 `Job`을 통해 부모에게 전파됩니다               |
| 현재 위치에서 코루틴을 호출합니다.                                               | 비동기 코루틴을 시작합니다                         |

`runBlocking`은 코루틴 스코프 함수와 많은 공통점을 가지는 것처럼 보일 수 있지만 큰 차이가 존재합니다.

`runBlocking`은 Blocking 함수로써 호출되면 해당 스레드를 차단하여 그 안에서 실행되는 코루틴이 완료될 때까지 기다립니다.
반면에 코루틴 스코프 함수는 suspending 함수로 코루틴이 일시 중단되어 다른 작업을 수행할 수 있는 기회를 제공한다는 차이점이 있습니다.

이로 인해 `runBlocking`은 코루틴 계층 구조의 최상단에 있어야 하며 주로 앱의 진입점이나 테스트 케이스 등에서 사용됩니다.
코루틴 스코프 함수는 일반적으로 이미 존재하는 코루틴 스코프 내에서 새로운 스코프를 생성하는 용도로 사용됩니다.

---

## withContext

`withContext`는 사용자가 특정 코루틴 컨텍스트를 명시적으로 설정할 수 있습니다.
이 컨텍스트는 부모 스코프의 컨텍스트를 오버라이드하며, 이로 인해 `withContext`내에서 실행되는 코루틴은 새로운 설정을 따릅니다.

`withContext(EmptyCoroutineContext)`를 사용하면, 부모 스코프의 컨텍스트를 그대로 이어받는 `coroutineScope`와 동일하게 작동됩니다.

```kotlin
fun CoroutineScope.log(text: String) {
    val name = this.coroutineContext[CoroutineName]?.name
    println("[$name] $text")
}

fun main() = runBlocking(CoroutineName("Parent")) {
    log("Before")

    withContext(CoroutineName("Child 1")) {
        delay(1000)
        log("Hello 1")
    }

    withContext(CoroutineName("Child 2")) {
        delay(1000)
        log("Hello 2")
    }

    log("After")
}
// [Parent] Before
// (1 sec)
// [Child 1] Hello 1
// (1 sec)
// [Child 2] Hello 2
// [Parent] After
```

`withContext`는 코드의 일부분에서 다른 코루틴 스코프를 설정하기 위해 자주 사용되며 보통 `Dispatchers` 컨텍스트와 함께 사용합니다.

```kotlin
launch(Dispatchers.Main) {
    view.showProgressBar()
    withContext(Dispatchers.IO) { fileRepository.saveData(data) }
    view.hideProgressBar()
}
```

`coroutineScope { ... }`와 `async { ... }.await()`는 매우 유사한 방식으로 동작하며,
비슷하게 `withContext(context) { ... }`와 `async(context) { ... }.await()`도 유사하게 동작합니다.

`async`는 코루틴 스코프가 필요하여 별도로 스코프를 지정해야 하기에 코드가 복잡해질 수 있습니다.
반면에 `coroutineScope`와 `withContext`는 이미 suspending 함수 내에서 실행되므로 자동으로 현재의 코루틴 스코프를 이어 받습니다.

이 때문에 코드의 가독성과 관리성을 위해서 `async`와 `await()`를 즉시 사용할 떄 특별한 경우가 아니라면 `coroutineScope` or `withContext`를 사용하게 좋습니다.

---

## supervisorScope

`supervisorScope`는 `coroutineScope`와 매우 유사하게 동작합니다.
외부 스코프로부터 `CoroutineScope`를 상속받고 지정된 suspend 블록을 실행합니다.

차이점이라면, 컨텍스트의 `Job`을 `SupervisorJob`으로 오버라이드하여 자식 코루틴이 예외를 발생시켜도 다른 코루틴들이 취소되지 않는다는 점 입니다.

```kotlin
fun main() = runBlocking {
    println("Before")

    supervisorScope {
        launch {
            delay(1000)
            throw Error()
        }
        launch {
            delay(2000)
            println("Done")
        }
    }

    println("After")
}
// Before
// 1s delay
// Exception
// 1s delay
// Done
// After
```

`supervisorScope`는 여러 독립적인 비동기 작업을 안전하고 효율적으로 관리할 수 있는 좋은 방법입니다.

```kotlin
suspend fun notifiyAnlaytics(actions: List<UserAction>) = supervisorScope {
    actions.forEach { action ->
        launch { notifyAnlaytics(action) }
    }
}
```

`async`로 시작한 코루틴의 결과를 받기 위해 `await()` 호출 시, 해당 코루틴이 예외로 종료되면 이 예외는 `await()`을 통해 다시 던져집니다.

`async`가 예외를 억제하도록 설정해도, 이는 부모 코루틴으로의 전파를 억제할 뿐, 실제로 `await()` 호출 시 예외가 다시 발생됩니다.
이 예외를 완전히 무시하려면 `await()` 호출을 `try-catch`로 감싸야 합니다.

따라서 `async`와 `await()` 사용 시 에외 처리를 충분히 고려해야 하며, `await()` 호출 주변에 `try-catch`를 사용하는 것이 좋습니다.

```kotlin
class ArticlesRepositoryComposite(
    private val articleRepositories: List<ArticleRepository>
) : ArticleRepository {
    override suspend fun fetchArticles(): List<Article> =
        supervisorScope {
            articleRepositories
                .map { async { it.fetchArticles() } }
                .mapNotNull {
                    try {
                        it.await()
                    } catch (e: Exception) {
                        e.printStackTrace()
                        null
                    }
                }
                .flatten()
                .sortedByDescending { it.publishedAt }
        }
}
```

`withContext(SupervisorJob())`를 사용하여 `supervisorScope`를 대체할 수 없습니다.

`withContext(SupervisorJob())`를 사용하면, `withContext`는 여전히 `Job`을 사용하게되고, `SupervisorJob()`은 그 `Job`의 부모가 됩니다.
이런 상속 구조로 인해 `withContext`의 자식 코루틴 중 하나가 실패하면, 다른 모든 자식 코루틴들도 실패하게 됩니다.  
즉, 예외가 발생된 자식 코루틴들은 예외를 `withContext`로 전파되기에 `SupervisorJob()`은 실질적으로 의미가 없게됩니다.

이러한 이유로 `withContext(SupervisorJob())`는 무의미하고 혼동을 줄 수 있으며, 좋은 방법이 아닙니다.

```kotlin
fun main() = runBlocking {
    println("Before")

    withContext(SupervisorJob()) {
        launch {
            delay(1000)
            throw Error()
        }
        launch {
            delay(2000)
            println("Done")
        }
    }

    println("After")
}
// Before
// 1s delay
// Exception
```

---

## withTimeout

`withTimeout`은 작업에 최대 실행 시간을 부여하는 역할을 하며, 시간이 지나면 작업을 강제 종료 시킵니다.

`withTimeout`에서 설정된 시간을 초과하게 되면 `TimeoutCancellationException`이 발생됩니다.  
(이 예외는 코루틴이 취소되었을 때 나타내는 `CancellationException`의 하위 타입입니다.)

`withTimeout`이 성공적으로 완료된 경우 블록 내에서 반환된 값을 반환합니다.

`withTimeout`은 특정 작업에 시간 제한을 두고 싶을 떄 유용하며 느린 네트워크 요청, 복잡한 계산 등에 시간 초과를 적용할 수 있어 앱의 반응성 유지에 도움이 됩니다.

```kotlin
suspend fun test(): Int = withTimeout(1500) {
    delay(1000)
    println("Still thinking")
    delay(1000)
    println("Done")
    42
}

suspend fun main() = coroutineScope {
    try {
        test()
    } catch (e: TimeoutCancellationException) {
        println("Too long")
    }

    delay(1000) // Extra timeout does not help, `test` body was cancelled
}
// 1s dealy
// Still thinking
// Too long
```

`withTimeout`은 테스트에 유용하며 어떤 함수(네트워크 호출, 복잡한 알고리즘 등)가 설정된 시간 보다 더 오래 걸리는지 또는 덜 걸리는지 테스트할 수 있습니다.

`runTest` 안에서 사용되면 가상의 시간에서 동작하게 됩니다. 또한 `runBlocking` 내부에서도 사용되어 특정 함수의 실행 시간을 제한할 수 있습니다.

```kotlin
class Test {
    @Test
    fun testTime2() = runTest {
        withTimeout(1000) {
            // something that should take less than 1s
            delay(900) // virtual time 
        }
    }

    @Test(expected = TimeoutCancellationException::class)
    fun testTime1() = runTest {
        withTimeout(1000) {
            // something that should take less than 1s
            delay(1100) // virtual time
        }
    }

    @Test
    fun testTime3() = runBlocking {
        withTimeout(1000) {
            // normal test, that should not take too long
            delay(900) // really waiting 0.9s
        }
    }
}
```

`TimeoutCancellationException`은 코루틴이 타임아웃에 도달했을 때 발생되는 특별한 형태의 `CancellationException` 입니다.
이 예외가 코루틴 빌더 내에서 발생되면, 해당 코루틴만 취소되고, 부모 코루틴에는 영향을 미치지 않습니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    launch { // 1
        launch { // 2, cancelled by it's parent
            delay(2000)
            println("Will not be printed")
        }
        withTimeout(1000) { // we cancel launch
            delay(1500)
        }
    }
    launch { // 3
        delay(2000)
        println("Done")
    }
}
// (2 sec)
// Done
```

`withTimeoutOrNull`는 함수 본문의 시간이 초과되면, 본문을 취소하고 `null`을 반환합니다.
`null`을 반환하므로, 이후 로직에서 이를 체크하여 적절한 조치(시간 초과 메시지 등)를 할 수 있습니다.

`withTimeoutOrNull`는 네트워크 호출, 데이터베이스 쿼리, I/O 작업 등에서 유용하며 이러한 작업들을 안전하게 처리할 수 있습니다.

```kotlin
suspend fun fetchUser(): User {
    // Runs forever
    while (true) {
        yield()
    }
}

suspend fun getUserOrNull(): User? = withTimeoutOrNull(5000) { fetchUser() }

suspend fun main() = coroutineScope {
    val user = getUserOrNull()
    println(user)
}
// 5s delay
// null
```

---

## Connecting coroutine scope functions

코루틴 스코프 함수(`withContext`, `withTimeout`, `supervisorScope`, `coroutineScope` 등)들은 자체적으로 특정한 기능을 제공합니다. 
그러나 때로는 여러 기능을 조합해야 할 수 있습니다. 이럴 때는 중첩해서 함수들을 사용할 수 있습니다.

예를 들어 타임아웃과 디스패처를 모두 설정하려면 아래와 같이 사용할 수 있습니다.

```kotlin
suspend fun calculateAnswerOrNull(): User? =
    withContext(Dispatchers.Default) {
       withTimeoutOrNull(1000) { calculateAnswer() }
    }
```

단, 코루틴 스코프 함수를 중첩해서 사용할 때 각 함수의 특성과 영향성을 잘 이해하고 사용해야 합니다.
예를 들어, `withContext`가 취소 불가능한 작업을 수행하는 경우, `withTimeoutOrNull`의 타임아웃은 효과가 없을 수 있습니다.

---

## Additional operations

아래 예시는 사용자 프로필을 표시한 후 Anayltics에 프로필을 표시했다는 것을 알리는 예시입니다.  
개발자들은 종종 같은 스코프에서 `launch`를 사용하여 수행합니다.

```kotlin
class ShowUserDataUseCase(
    private val repo: UserDataRepository,
    private val view: UserDataView
) {
    suspend fun showUserData() = coroutineScope { 
        val name = async { repo.getUserName() } 
        val friends = async { repo.getFriends() }
        val profile = async { repo.getUserProfile() }
        val user = User(
            name = name.await(),
            friends = friends.await(),
            profile = profile.await()
        )
   
        view.show(user)
        launch { repo.notifyProfileShown() }
    }
}
```

이러한 접근 방법은 아래와 같은 문제점이 있습니다.

`coroutineScope` 내에서 `launch`로 코루틴을 시작하면, `coroutineScope` 블록은 내부 코루틴이 모두 완료될 때까지 대기합니다.

만약 아래 코드와 같이 `progressBar`를 표시하고 있다면 `notifyProfileShown()`와 같이 추가 작업이 끝나지 않으면 사용자는 그 작업이 완료될 때까지 업데이트 된 View를 볼 수 없습니다.

```kotlin
fun onCreate() {
    viewModelScope.launch {
        _progressBar.value = true
        showUserData()
        _progressBar.value = false
    }
}
```

그 다음 문제로는 취소(Cancellation) 문제 입니다.

코루틴은 기본적으로 예외가 발생되면 다른 작업을 취소하도록 설계되어 있습니다.
필수적인 작업에 대해서는 이러한 부분이 좋은 방식이지만, 만약 `getUserProfile`에서 예외가 발생되면 `getUserName`과 `getFriends`의 응답이 무용지물이므로 이들을 취소해야 합니다.

그러나 `notifyProfileShown()` 호출이 실패했다고 하여 위 작업들을 취소하는 것은 크게 의미가 없는 행동입니다.

그렇기에 메인 로직에 영향을 미치지 않는 추가적인 작업이 있을 때에는 아래와 같이 별도의 스코프를 생성하여 시작하는 것이 좋습니다. 

      val analyticsScope = CoroutineScope(SupervisorJob())

이처럼 생성된 스코프를 단위 테스트하고 제어하기 위해서는 생성자를 통해 주입하는 것이 좋은 방식입니다.

```kotlin
class ShowUserDataUseCase(
    private val repo: UserDataRepository,
    private val view: UserDataView,
    private val analyticsScope: CoroutineScope
) {
    suspend fun showUserData() = coroutineScope { 
        val name = async { repo.getUserName() } 
        val friends = async { repo.getFriends() }
        val profile = async { repo.getUserProfile() }
        val user = User(
            name = name.await(),
            friends = friends.await(),
            profile = profile.await()
        )
   
        view.show(user)
        analyticsScope.launch { repo.notifyProfileShown() }
    }
}
```

위처럼 스코프를 명시적으로 주입하면 해당 클래스가 독립적으로 비동기 작업을 시작할 수 있다는 것이 명확해지기에 다른 개발자나 코드 리뷰어가 이 클래스의 동작을 더 쉽게 이해할 수 있습니다. 

이처럼 스코프를 전달하여 해당 클래스가 독립적인 호출을 시작함을 알 수 있으면 suspending 함수가 시작한 모든 작업을 기다리지 않을 수 있다는 것 알 수 있습니다.  
또한 스코프가 전달되지 않으면 suspending 함수는 시작한 모든 작업이 완료될 때까지 완료되지 않음을 예상할 수 있습니다.