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