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