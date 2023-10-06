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