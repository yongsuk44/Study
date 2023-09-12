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