# Best practices

## Don't use async with an immediate await

`async`와 `await`를 동시에 사용하는 것은 피해야 합니다.  
이렇게 하면 `async`로 시작된 작업이 별도의 동시성을 제공하지 않고, 
즉시 `await`로 기다려야 하기에 별도의 코루틴을 시작하는 의미가 사라집니다.

```kotlin
// Don't
suspend fun getUser(): User = coroutineScope {
    val user = async { repo.getUser() }.await()
    user.toUser()
}
// Do
suspend fun getUser(): User {
    val user = repo.getUser()
    return user.toUser()
}
```

비동기 작업 시 다음과 같은 주의사항이 있습니다.

1. 스코프가 필요한 경우 `async { ... }.await()` 대신 `coroutineScope { ... }`를 사용하는 것이 좋습니다.
2. 컨텍스트 변경이 필요한 경우 `withContext`를 사용하는 것이 좋습니다.
3. 여러 개의 `async` 작업 시 가독성을 위해 모든 작업에 `async`를 적용하는 것이 좋습니다.

```kotlin
fun showNews() {
    viewModelScope.launch {
        val config = async { getConfigFromApi() }
        val news = async { getNewsFromApi(config.await()) }
        val user = async { getUserFromApi() }
        
        view.showNews(user.await(), news.await())
    }
}
```

----

## Use coroutineScope instead of withContext(EmptyCoroutineContext)

코루틴 사용 시 `withContext`와 `coroutineScope` 사이에서 선택해야 할 수 잇습니다.

`withContext`는 컨텍스트를 오버라이드 할 수 있지만, 컨텍스트 변경이 필요하지 않는 경우 `withContext(EmptyCoroutineContext)`를 사용하는 경우에는
`coroutineScope`를 사용하는 것이 코드가 더 명확해지고 의미가 분명해집니다.

----

## Use awaitAll

비동기 작업을 기다릴 때 `awaitAll`을 사용 시 첫 번째 `async` 작업에서 예외가 발생하면 `await`를 중지하기에, 불필요한 대기 시간을 줄이고 문제를 빠르게 파악할 수 있습니다.  
반면에 `map { it.await() }` 사용 시 각 작업을 순차적으로 기다리게 되므로, 하나의 작업에서 문제가 발생하면 그 이전의 모든 작업이 완료될 떄까지 대기해야 합니다.

따라서 `awaitAll`을 사용하면 시스템 리소스를 더 효율적으로 활용할 수 있습니다.