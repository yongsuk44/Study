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