# Common use cases

코루틴을 사용한 Application 구조는 다음 3가지 주요 계층으로 구성될 수 있습니다.

- Data / Adapters Layer : 데이터를 관리하거나 다른 시스템과의 상호 작용을 담당합니다.
- Domain Layer : 핵심 비지니스 로직이 이루어지는 부분입니다.
- Presentation / UI Layer : 사용자와 상호 작용하는 UI 계층 입니다. 주로 코루틴의 실행 결과를 처리하게 됩니다.

---

## Data / Adapters layer

데이터의 저장, 추출, 변환 등과 같은 작업들이 이루어지며,
현재 JVM에서 사용되는 많은 라이브러리들이 코루틴을 지원하기에 이 계층의 구현은 비교적 간단해졌습니다.

Retrofit과 같은 라이브러리는 네트워크 요청을 쉽게 구현할 수 있게 해주며,
Retrofit은 코루틴을 위한 일시 중지 함수를 기본적으로 지원하기 때문에 별도의 작업 없이 비동기 네트워크 요청을 구현할 수 있습니다.

이는 요청 함수에 `suspend` 키워드만 추가하면 해당 함수는 차단이 아닌 일시 정지 함수로 변환되어 코루틴에서 안전하게 호출될 수 있습니다.

```kotlin
class GithubApi {
    @GET("orgs/{org}/repos?per_page=100")
    suspend fun getOrgRepos(
        @Path("org") org: String
    ): List<Repo>
}
```

또 다른 예로 Room도 `suspend` 키워드를 통해 데이터베이스 작업을 비동기적으로 수행할 수 있게 지원합니다.   
이는 쿼리와 같은 장시간 실행 작업을 메인 스레드에서 차단하지 않고 처리할 수 있게 해줍니다.  
또한 특정 테이블이나 쿼리 결과의 변경을 관찰하기 위해 `Flow`를 지원하여 쉬운 반응형 UI 구현을 도와줍니다.

```kotlin
@Dao
interface LocationDao {
    @Query("DELETE FROM location_table")
    suspend fun deleteLocations()

    @Query("SELECT * FROM location_table")
    fun getAll(): Flow<List<Location>>
}
```

### callback functions

코루틴을 지원하지 않고 콜백 함수를 사용하도록 강제하는 라이브러리를 사용해야 하는 경우,
`suspendCancellableCoroutine`을 사용하여 해당 콜백 함수를 일시 중지 함수로 변환할 수 있습니다.

콜백이 호출될 때, 해당 코루틴은 `resume`을 통해 재개되며,
콜백이 취소 가능한 경우 코루틴이 취소되었을 때 해당 콜백도 취소되도록 `invokeOnCancellation`을 통해 취소 로직을 정의할 수 있습니다.

```kotlin
suspend fun requestNews(): News {
    return suspendCancellableCoroutine<News> { continuation ->
        val call = requestNewsApi { news -> continuation.resume(news) }

        continuation.invokeOnCancellation { call.cancel() }
    }
}
```

성공과 실패에 대해 별도의 함수를 설정하는 콜백 함수는 다음과 같이 `Result`를 통해 처리할 수 있습니다.

```kotlin
suspend fun requestNews(): News {
    return suspendCancellableCoroutine<News> { continuation ->
        val call = requestNewsApi(
            onSucess = { news -> continuation.resume(Result.success(news)) },
            onError = { error -> continuation.resume(Result.failure(error)) }
        )

        continuation.invokeOnCancellation { call.cancel() }
    }
}
```

또 다른 옵션으로 함수 반환 타입을 `null`을 허용하여 `null` 값으로 코루틴을 재개할 수 있습니다.

```kotlin
suspend fun requestNews(): News? {
    return suspendCancellableCoroutine<News> { continuation ->
        val call = requestNewsApi(
            onSucess = { news -> continuation.resume(news) },
            onError = { error -> continuation.resume(null) }
        )

        continuation.invokeOnCancellation { call.cancel() }
    }
}
```

마지막으로 콜백이 성공적으로 값을 반환한 경우 해당 값을 통해 코루틴을 재개하고  
콜백이 실패하거나 오류를 반환하는 경우 적절한 예외를 발생시켜 코루틴을 중단시킬 수 있습니다.

```kotlin
suspend fun requestNews(): News {
    return suspendCancellableCoroutine<News> { continuation ->
        val call = requestNewsApi(
            onSucess = { news -> continuation.resume(news) },
            onError = { error -> continuation.resumeWithException(error) }
        )

        continuation.invokeOnCancellation { call.cancel() }
    }
}
```

### Blocking functions

코루틴의 핵심 원칙 중 하나는 스레드를 효과적으로 활용하여 동시성을 처리하는 것입니다.  
이 원칙을 지키기 위해 일시 중지 함수는 백그라운드에서 블로킹 없이 실행됩니다.  
그러나 블로킹 함수를 호출하면 해당 스레드가 차단되어 다른 코루틴 작업에 사용할 수 없게 됩니다.

특히, Android의 `Dispatcher.Main`에서 스레드를 차단하면 ANR 상태에 빠질 수 있습니다.  
또한 `Dispatchers.Default`에서 스레드를 차단하면 프로세서를 효율적으로 사용하지 못하게 됩니다.  
이 때문에 디스패처를 먼저 지정하지 않고 스레드 블로킹을 하면 안됩니다.

따라서 블로킹 호출이 필요할 때는 `withContext`를 사용하여 디스패처를 지정해야 합니다.  
대부분의 경우 저장소를 구현할 때 `Dispatchers.IO`를 사용합니다.

```kotlin
class DiscSaveRepository(
    private val discReader: DiscReader
): SaveRepository {
    override suspend fun loadSave(name: String): SaveData =
        withContext(Dispatchers.IO) {
            discReader.read("save/$name")
        }
}
```

`Dispatchers.IO`는 I/O 작업에 최적화된 디스패처로 스레드 풀 크기가 기본적으로 64개로 제한되어 있습니다.  
이는 대부분의 작업에 충분하지만, 블로킹 호출을 동시에 여러번 발생하는 높은 부하 시나리오에서는 제한될 수 있습니다.

이 문제를 해결하기 위해 `limitedParallelism`을 사용하면 특정 수의 최대 스레드를 사용하여 사용자 정의 디스패처를 생성할 수 있습니다.  
이를 통해 주어진 상황에 맞게 디스패처의 스레드 풀 크기를 조정할 수 있습니다.

```kotlin
class LibraryGoogleAccountVerifier: GoogleAccountVerifier {
    private val dispatcher = Dispatchers.IO.limitParallelism(100)
    
    private var verifier = 
        GoogleIdTokenVerifier.Builder(..., ...)
            .setAudience(...)
            .build()
    
    override suspend fun getUserData(googleToken: String): GoogleUserData? =
        withContext(dispatcher) {
            verifier.verify(googleToken)
                ?.payload
                ?.let {
                    GoogleUserData(
                        email = it.email,
                        name = it.getString("given_name"),
                        surname = it.getString("family_name"),
                        pictureUrl = it.getString("picture")
                    )
                }
        }
}
```

하지만 `Dispatchers.IO`에 너무 많은 코루틴이 동시에 할당되면, 모든 스레드가 사용될 위험이 있습니다.   
이 때문에 특정 작업에 많은 스레드를 사용할 것으로 예상되면, `Dispatchers.IO`와는 별개로 제한이 잇는 디스패처를 사용하여 스레드 사용을 관리하는 것이 좋습니다.

라이브러리를 만들 때 다양한 사용자와 상황에서 어떻게 활용될 지 예측하기가 어렵습니다.  
따라서 최적의 성능과 자원 사용을 보장하기 위해 독립적인 스레드 풀을 가진 디스패처를 사용하는 것이 좋습니다.  
하지만 이 디스패처의 스레드 수를 결정하는 것은 중요한 결정이므로, 신중하게 고려해야 합니다.

스레드 수의 제한이 너무 작은 경우 스레드가 부족하여 여러 코루틴이 동시에 실행되지 못하고 대기 상태에 머물 수 있어 성능 저하를 초래할 수 있고,
스레드 수의 제한이 너무 큰 경우 활성 스레드의 수가 많아지게 되어, 메모리와 CPU 자원이 증가함에 따라 시스템 전반적인 영향을 줄 수 있습니다.

```kotlin
class CertificateGenerator {
    private val dispatcher = Dispatchers.Default.limitParallelism(4)
    
    suspend fun generateCertificate(): UserData =
        withContext(dispatcher) {
            Runtime.getRuntime().exec("generateCertificate "+ data.toArgs())
        }
}
```

이에 코루틴은 다양한 디스패처를 제공하여 특정 타입의 작업을 적절한 스레드에서 실행할 수 있도록 지원합니다.

```kotlin
suspend fun calculateModel() = 
    withContext(Dispatchers.Default) {
        model.fit(
            dataset = newTrain,
            epochs = 10,
            verbose = false,
            batchSize = 128
        )
    }

suspend fun setUserName(name: String) = 
    withContext(Dispatchers.Main.immediate) {
        userNameView.text = name
    }
```

### Observing with Flow

일시 중지 함수는 단일 값을 생성하거나 가져오는 과정에 적합하지만, 여러 값을 생성하거나 가져오는 경우에는 `Flow`를 사용해야 합니다.

Room 라이브러리에서는 단일 데이터베이스 작업을 수행하기 위해 `suspend`가 붙은 함수를 사용하고,
테이블의 변경 사항을 관찰하기 위해 `Flow` 타입을 사용합니다.

```kotlin
@Dao
interface LocationDao {
    @Query("DELETE FROM location_table")
    suspend fun deleteLocations()
    
    @Query("SELECT * FROM location_table")
    fun getAll(): Flow<List<Location>>
}
```

API에서 단일 값을 가져올 땐 `suspend` 함수 사용시 최선이지만,  실시간 메시지 수신과 같은 지속적인 데이터 스트림에도 `Flow`를 사용하는 것이 좋습니다.  
`callbackFlow`나 `channelFlow`는 기존의 콜백 방식의 API나 이벤트를 `Flow` 형태로 변환할 때 사용됩니다.

```kotlin
fun listenMessages(): Flow<List<Message>> = callbackFlow {
    socket.on("NewMessage") { args -> trySendBlocking(args.toMessage()) }
    awaitClose()
}
```

`Flow`는 버튼 클릭이나 텍스트 변경과 같은 UI 이벤트를 관찰하는데도 효과적입니다.

```kotlin
fun EditText.listenTextChange(): Flow<String> = callbackFlow {
    val watcher = doAfterTextChanged { text -> trySendBlocking(text.toString()) }
    awaitClose { removeTextChangedListener(watcher) }
}
```

또한 다른 콜백 함수들에도 사용될 수 있으며, 콜백이 여러 값을 생성할 수 있을 때 사용될 수 있습니다.

```kotlin
fun flowFrom(api: CallbackBasedApi): Flow<T> = callbackFlow {
    
    val callback = object : Callback {
        override fun onNextValue(value: T) { 
            trySendBlocking(value)
        }
        
        override fun onApiError(error: Throwable) {
            cancel(CancellationException("Api Error", error))
        }
        
        override fun onCompleted() {
            channel.close()
        }
    }
    
    api.register(callback)
    awaitClose { api.unregister(callback) }
}
```

--------------------------------------------------------------------------------------------------

## Domain layer

도메인 계층은 비지니스 로직이 구현되는 곳이므로, 여기에서 UseCase, Services 등을 정의합니다.    
또한 도메인 계층에서는 코루틴 스코프 객체에 직접 작업을 수행하거나, 일시 중단 함수를 외부에 노출하는 것을 피해야 합니다.  
이러한 작업은 프레젠테이션 계층에서 수행되어야 합니다.

실제로 도메인 계층에서 비지니스 로직을 구현할 때 여러 작업을 연속적으로 수행해야 할 경우가 많기 때문에 
다른 suspend 함수를 호출하는 suspend 함수를 주로 가지고 있습니다.

```kotlin
class NetworkUserRepository(
    private val api: UserApi
): UserRepository {
    override suspend fun getUser(): User = api.getUser().toDomainUser()
}

class NetworkNewsService(
    private val newsRepo: NewsRepository,
    private val settings: SettingsRepository
) {
    suspend fun getNews(): List<News> = 
        newsRepo.getNews().map { it.toDomainNews() }
    
    suspend fun getNewsSummary(): List<News> {
        val type = settings.getNewsSummaryType()
        return newsRepo.getNewsSummary(type)
    }
}
```

### Concurrent calls

동시에 여러 프로세스를 실행하려면, `coroutineScope`를 사용하여 함수 본문을 감싸고 해당 스코프 내에서 시작된 모든 코루틴이 완료될 떄까지 기다릴 수 있습니다.
병렬로 실행되어야 하는 각 프로세스는 `async` 빌더를 사용하여 비동기 작업을 시작할 수 있습니다.

```kotlin
suspend fun produceCurrentUser(): User = coroutineScope {
    val profile = async { repo.getProfile() }
    val friends = async { repo.getFriends() }
    User(
        profile = profile.await(),
        friends = friends.await()
    )
}
```

병렬 처리 시 주의할 점으로 기존에 있던 취소, 예외 처리, 컨텍스트 전달 등의 메커니즘은 그대로 유지되어야 합니다.  
예를 들어 `produceCurrentUserSeq`는 순차적으로 동작하는 반면, `produceCurrentUserPar`는 병렬로 동작합니다.

이외의 모든 기능은 동일하게 작동해야 합니다.

```kotlin
suspend fun produceCurrentUserSeq(): User {
    val profile = repo.getProfile()
    val friends = repo.getFriends()
    return User(profile = profile, friends = friends)
}

suspend fun produceCurrentUserPar(): User = coroutineScope {
    val profile = async { repo.getProfile() }
    val friends = async { repo.getFriends() }
    User(profile = profile.await(), friends = friends.await())
}
```

비동기 작업을 2개 시작할 때 선택할 수 있는 2가지 방법이 있습니다.  
첫 번째는 각 작업에 `async`를 사용하는 것이고, 두 번째는 하나의 작업만 `async`를 사용하고 다른 하나를 같은 코루틴에서 실행하는것 입니다.

첫 번째 방법은 코드의 가독성을 높이는 장점을 지니고, 두 번째 방법은 효율성이 높다는 장점이 있습니다.  
어떤 방법을 선택할지는 개발자나 프로젝트의 특성에 따라 다를 수 있으므로 둘 중 하나 선택하면 됩니다.

```kotlin
suspend fun produceCurrentUserPar(): User = coroutineScope {
    val profile = async { repo.getProfile() }
    val friends = repo.getFriends() 
    User(profile = profile.await(), friends = friends)
}

suspend fun getArticlesForUser(
    userToken: String
): List<ArticleJson> = coroutineScope {
    val articles = async { articleRepository.getArticles() }
    val user = userService.getUser(userToken)
    
    articles.await()
        .filter { canSeeOnList(user, it) }
        .map { toArticleJosn(it) }
}
```

`async`를 컬렉션 함수와 함께 사용하여 리스트의 각 요소에 대한 비동기 프로세스를 시작할 수 있습니다.  
이러한 경우 `awaitAll()`을 사용하여 모든 비동기 작업의 완료를 기다리는 것이 좋습니다.

`awaitAll`은 여러 `Deferred` 객체의 완료를 동시에 기다리고, 모든 작업이 완료되면 그 결과를 한 번에 가져옵니다.

```kotlin
suspend fun getOffers(
    categories: List<Category>
): List<Offer> = coroutineScope { 
    categories
        .map { async { api.requestOffers(it) } }
        .awaitAll()
        .flatten()
}
```

만약 동시 호출 수를 제한하려면 리스트를 `Flow`로 변환하고 `flatMapMerge`를 사용하는 방법이 있습니다.  
`flatMapMerge`는 `concurrency` 파라미터를 통해 동시 호출의 수를 제한할 수 있습니다.

```kotlin
fun getOffers(
    categories: List<Category>
): Flow<List<Offer>> = 
    categories.asFlow()
        .flatMapMerge(concurrency = 20) {
            suspend { api.requestOffers(it) }.asFlow()
            // or flow { emit(api.requestOffers(it)) }
        }
        
```

`coroutineScope`를 사용할 때는 자식 코루틴에서 발생한 예외가 전체 코루틴을 중단시키고 다른 자식 코루틴들을 취소시키는 점을 주의해야 합니다.  
이는 모든 작업이 독립적으로 수행되어야 하는 경우에 문제가 될 수 있기에 이런 상황에서 `supervisorScope`를 사용하면 됩니다.

`supervisorScope`는 자식 코루틴에서 발생하는 예외를 무시하므로, 독립적인 병렬 작업을 안전하게 수행할 수 있습니다.

```kotlin
suspend fun notifyAnalytics(actions: List<UserAction>) = supervisorScope {
    actions.forEach { action -> 
        launch { sendNotifyAnalytics(action) } 
    }
}
```

작업의 실행 시간을 제한하고 싶을 때, `withTimeout` 또는 `withTimeoutOrNull` 함수를 사용할 수 있습니다.  
두 함수 모두 지정된 시간을 초과하면 작업을 자동으로 취소합니다.

`withTimeout`은 시간 초과시 예외를 발생시키고 `withTimeoutOrNull`은 `null`을 반환합니다.

```kotlin
suspend fun getUserOrNull(): User? = 
    withTimeoutOrNull(5000) { fetchUser() }
```

### Flow transformations

`Flow` 처리 시 주로 사용하는 함수는 `map`, `filter`, `onEach` 등의 기본적힌 함수입니다.  
`map`은 각 요소를 변환, `filter`는 조건에 맞는 요소만 선택, `onEach`는 각 요소에 작업을 수행하는 등의 역할을 합니다.

특별한 상황에서 유용한 함수로는 `scan`과 `flatMapMerge`가 있습니다.  
`scan`은 누적 결과를 생성하고, `flatMapMerge`는 병렬 처리를 가능하게 합니다.

```kotlin
class UserStateProvider(
    private val userRepository: UserRepository
) {
    fun userStateFlow(): Flow<User> = userRepository
        .observeUserChanges()
        .filter { it.isSignificantChange }
        .scan(userRepository.currentUser()) { user, update -> 
            user.with(update)
        }
        .map { it.toDomainUser() }
}
```

두 개의 `Flow`를 합치고 싶을 때 `merge`, `zip`, `combine`과 같은 함수를 사용할 수 있습니다. 

`merge`는 각 `Flow`에서 발생하는 아이템을 순서에 상관없이 하나의 `Flow`로 합칩니다.  
`zip`은 두 `Flow`의 아이템을 쌍으로 묶어서 하나의 아이템으로 만듭니다.  
`combine`은 각 `Flow`의 최신 아이템을 사용하여 새로운 아이템을 생성합니다. 

```kotlin
class ArticlesProvider(
    private val blog: ArticleRepository,
    private val news: ArticleRepository
) {
    fun observeArticles(): Flow<Article> = merge(
        blog.observeArticles().map { it.toArticle() },
        news.observeArticles().map { it.toArticle() }
    )
}

class NotificationStatusProvider(
    private val userStateProvider: UserStateProvider,
    private val notificationsProvider: NotificationsProvider,
    private val statusFactory: NotificationStatusFactory
) {
    fun notificationStatusFlow(): NotificationStatus =
        notificationsProvider.observeNotifications()
            .filter { it.status == Notification.UNSEEN }
            .combine(userStateProvider.userStateFlow()) { notifications, user ->
                statusFactory.produce(notifications, user)
            }
}
```

하나의 `Flow`를 여러 코루틴에서 관찰하려면 `SharedFlow`를 사용하는 것이 좋습니다.  
이를 위해 `shareIn`을 사용하여 특정 스코프 내에서 `Flow`를 공유하게 해줍니다.

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

--------------------------------------------------------------------------------------------------

## Presentation / UI layer

코루틴은 주로 Presentation 계층에서 실행됩니다.

안드로이드에서는 WorkManager를 통해 백그라운드 작업을 쉽게 예약하고 관리할 수 있습니다.  
이떄 코루틴은 `CoroutineWorker`를 제공하며 이 클래스의 `doWork`를 구현하면 해당 작업이 어떤일을 해야 하는지 지정할 수 있습니다.

여기서 `doWork`는 suspend 함수로 라이브러리 자체가 이 함수를 코루틴에서 자동으로 실행해주기 때문에, 별도로 코루틴을 시작할 필요가 없습니다.

```kotlin
class CoroutineDownloadWorker(
    context: Context,
    params: WorkerParameters
): CoroutineWork(context, params) {
    
    override suspend fun doWork(): Result {
        val data = downloadSyncronously()
        saveData(data)
        return Result.success()
    }
}
```

그러나 대부분의 상황에서 라이브러리나 프레임워크가 코루틴을 자동으로 시작하지 않습니다.  
이런 경우 `scope` 객체에 `launch` 함수를 사용하여 코루틴을 시작합니다. 

안드로이드에서는 `lifecycle-viewmodel-ktx` 라이브러리를 통해 `viewModelScope`나 `lifecycleScope`를 대부분의 경우에 사용할 수 있습니다.  
이를 통해 코루틴의 생명주기를 안드로이드의 생명주기와 쉽게 연동할 수 있어 비동기 작업을 더욱 효율적으로 관리할 수 있습니다.

```kotlin
class UserProfileViewModel(
    private val loadProfileUseCase: LoadProfileUseCase,
    private val updateProfileUseCase: UpdateProfileUseCase
) {
    private val userProfile = MutableSharedFlow<UserProfileData>()
    
    val userName: Flow<String> = userProfile.map { it.name }
    val userSurName: Flow<String> = userProfile.map { it.surname }
    
    fun onCreate() {
        viewModelScope.launch {
            val userProfileData = loadProfileUseCase.execute()
            userProfile.value = userProfileData
        }
    }
    
    fun onNameChange(newName: String) { 
        viewModelScope.launch {
            val newProfileData = userProfile.value.copy(name = newName)
            userProfile.value = newProfileData
            updateProfileUseCase.execute(newProfileData)
        }
    }
}
```

### Creating custom scope

라이브러리나 클래스가 코루틴을 자동으로 시작하지 않는 상황에서는 개발자가 직접 `scope`를 생성하여 코루틴을 시작할 수 있습니다.  
이 방법은 라이브러리나 프레임워크의 지원이 없는 상황에서 유용하며 이를 통해 코루틴의 생명주기를 더욱 세밀하게 관리할 수 있습니다.

```kotlin
class NotificationSender(
    private val client: NotificationClient,
    private val notificationScope: CoroutineScope
) {
    fun sendNotifications(notifications: List<Notification>) {
        notifications.forEach { notification ->
            notificationScope.launch { 
                client.send(notification)
            }
        }
    }
}

class LatestNewsViewModel(
    private val newsRepo: NewsRepository
): BaseViewModel() {
    private val _uiState = MutableStateFlow<NewsState>(LoadingNews)
    val uiState: StateFlow<NewsState> = _uiState
    
    fun create() {
        scope.alunch {
            _uiState.value = NewsLoaded(newsRepo.getNews())
        }
    }
}
```

`CoroutineScope` 정의 시 디스패처나 예외 핸들러를 설정할 수 있습니다.  
이러한 스코프는 안드로이드에서는 특정 조건 아래 자식 코루틴을 취소할 수 있습니다.

예를 들어 ViewModel은 자신이 클리어될 때 스코프를 취소하고, WorkManager는 관련 작업이 취소될 때 스코프를 취소합니다.  
이렇게 해서 코루틴의 생명주기와 예외 상황을 효과적으로 관리할 수 있습니다.

```kotlin
abstract class BaseViewModel: ViewModel() {
    private val _failure = MutableLiveData<Throwable>()
    val failure: LiveData<Throwable> = _failure
    
    private val exceptionHandler = CoroutineExceptionHandler { _, exception ->
        _failure.value = exception
    }
    
    private val ctx = Dispatchers.Main + exceptionHandler + SupervisorJob()
    protected val scope = CoroutineScope(ctx)
    
    override fun onCleared() {
        ctx.cancelChildren()
    }
}
```