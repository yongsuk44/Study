# Constructing a coroutine scope

## CoroutineScope factory function

`CoroutineScope`는 단일 프로퍼티 `coroutineContext`를 지닌 인터페이스 입니다.

```kotlin
interface CoroutineScope {
    val coroutineContext: CoroutineContext
}
```

따라서 이 인터페이스를 구현하는 클래스를 정의하고 그 안에서 직접 코루틴 빌더를 호출할 수 있습니다.

```kotlin
class SomeTest : CoroutineScope {
    override val coroutinecontext: Coroutinecontext = Job()

    fun onStart() {
        launch {
            // ...
        }
    }
}
```

그러나 위와 같이 클래스가 직접 `CoroutineScope`를 구현하면 클래스 안에서 다른 `CoroutineScope` 메서드를 직접 호출할 수 있습니다.   
만약 해당 스코프에 `cancel`을 호출하면 전체 스코프에 있는 모든 코루틴들이 취소될 수 있는 문제가 발생될 수 있습니다.

이로 인해 대부분의 경우, 프로퍼티로 코루틴 스코프를 객체로 가지고 있고 이 프로퍼티를 활용하여 코루틴 빌더를 호출하는 방식을 선호합니다.

```kotlin
class SomeTest {
    val scope: CoroutineScope = ...

    fun onStart() {
        scope.launch {

        }
    }
}
```

코루틴 스코프 객체를 생성하는 가장 간편한 방법은 `CoroutineScope` 팩토리 함수를 사용하는 것입니다.

이 함수는 제공된 컨텍스트로 스코프를 생성합니다. (만약 컨텍스트에 `Job`이 없다면 구조화된 동시성을 위해 `Job`을 생성합니다.)

```kotlin
fun CoroutineScope(
    context: CoroutineContext
): CoroutineScope = ContextScope(
    if (context[Job] != null) context
    else context + Job()
)

internal class ContextScope(
    context: CoroutineContext
) : CoroutineScope {
    override val coroutinecontext: CoroutineContext = context
    override fun toString(): String = "CoroutineScope(coroutineContext = $coroutineContext)"
}
```

---

## Constructing a scope on Android

대부분의 안드로이드 앱에서는 MVVM 혹은 MVP 패턴를 사용하고, 이 패턴에서 ViewModel 또는 Presenter에서 주로 코루틴을 시작합니다.  
또한 코루틴이 Fragment, Activity에서도 시작될 수 있습니다.

이처럼 안드로이드에서 코루틴이 어디에서 시작되든, 그 구성 방식은 대체로 동일합니다.

    UseCase 혹은 Repository와 같은 다른 레이어에서는 보통 suspending 함수만을 사용합니다.

아래 코드를 보면, `MainViewModel`의 `onCreate`에서 어떤 데이터를 가져와야 한다고 가정할 때,
데이터를 얻기 위해선 어떤 스코프에서 호출되는 코루틴에서 데이터를 불러와야 할 것입니다.

여기서 `BaseViewModel`에서 공통적으로 사용될 코루틴 스코프를 생성하고 관리하는것은 좋은 방법이 됩니다.
이렇게 하면 중복된 코드를 줄이고 모든 ViewModel에서 동일한 생명주기 관리 방식을 사용할 수 있습니다.

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope = CoroutineScope(TODO())
}

class MainViewModel(
    private val userRepo: UserRepository,
    private val newsRepo: NewsRepository
) : BaseViewModel() {
    fun onCreate() {
        scope.launch {
            val user = userRepo.getUser()
            view.showUserData(user)
        }

        scope.launch {
            val news = newsRepo.getNews()
        }
    }
}
```

이제 코루틴 스코프에 대한 컨텍스트를 정의해보면,
안드로이드에서는 많은 함수가 메인 스레드에서 호출되어야 하므로 `Dispatchers.Main`이 기본 디스패처로 가장 좋은 옵션으로 간주됩니다.

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope = CoroutineScope(Dispatchers.Main)
}
```

그 다음, 사용자가 화면을 벗어날 때나 `onDestory`(ViewModel의 경우 `onCleared`)가 호출될 때 모든 미완료 프로세스를 취소하는 것은 일반적입니다.
이처럼 모든 미완료 프로세스를 취소하려면 스코프를 취소하게 만들어주어야 합니다.

스코프를 취소 가능하게 만드려면 `Job` 컨텍스트를 가지고 있어야 합니다. (`Job`을 추가하지 않아도 `CoroutineScope` 함수가 자동으로 추가하지만 명시적으로 추가하는 것이 더 명확합니다.)
그 다음 `onCleared`에서 스코프를 취소하면 됩니다.

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope = CoroutineScope(Dispatchers.Main + Job())

    override fun onCleared() {
        scope.cancel()
    }
}
```

더 나아가서 전체 스코프를 취소하는 대신 자식 코루틴만 취소하면, `ViewModel`이 여전히 활성 상태일 때 새로운 코루틴을 시작할 수 있습니다.

이러한 방식은 특정 작업을 취소하고 ViewModel에서 다른 작업을 계속 수행할 수 있는 유연성을 제공하게 됩니다.

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope = CoroutineScope(Dispatchers.Main + Job())

    // Job 객체가 자식 코루틴만을 추적하기에 `cancelChildren()`을 통해 자식 코루틴만 취소 가능
    override fun onCleared() {
        scope.coroutineContext.cancelChildren()
    }
}
```

추가로 이 스코프에서 시작된 다양한 코루틴이 독립적이길 원한다면, `Job`을 사용하는 대신 `SupervisorJob`을 사용할 수 있습니다.

`Job`은 어떤 자식 코루틴이 오류로 인해 취소되면 다른 코루틴도 모두 취소되지만,  
`SupervisorJob`은 한 작업에서 발생한 오류가 다른 작업에 영향을 주지 않기에 독립적으로 동작할 수 있습니다.

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())

    override fun onCleared() {
        scope.coroutineContext.cancelChildren()
    }
}
```

마지막으로 코루틴에서 처리되지 않은 예외에 대해서 기본적인 예외 처리 방식을 `CoroutineExceptionHandler`를 통해 정의합니다.

`CoroutineExceptionHandler`은 코루틴이 던진 처리되지 않은 예외를 catch하고 특정 작업을 수행하며
이 핸들러는 코루틴 컨텍스트의 일부로 추가될 수 있습니다.

이처럼 예외 처리 로직을 한 곳에서 관리하면 코드 재사용성이 높아지고 일관된 사용자 경험을 제공하게 할 수 있습니다.

```kotlin
abstract class BaseViewModel(
    private val onError: (Throwable) -> Unit
) : ViewModel() {
    private val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
        onError(throwable)
    }

    private val context = Dispatchers.Main + SupervisorJob() + exceptionHandler
    protected val scope = CoroutineScope(context)

    override fun onCleared() {
        context.cancelChildren()
    }
}
```

위 코드에 대한 대안으로 예외를 `LiveData`로 보유하고 이를 다른 View 요소에서 관찰하도록 할 수 있습니다.

```kotlin
abstract class BaseViewModel : ViewModel() {
    private val _failure: MutableLiveData<Throwable> = MutableLiveData()
    val failure: LiveData<Throwable> = _failure

    private val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
        _failure.value = throwable
    }

    private val context = Dispatchers.Main + SupervisorJob() + exceptionHandler
    private val scope = CoroutineScope(context)

    override fun onCleared() {
        context.cancelChildren()
    }
}
```

--- 

## viewModelScope and lifecycleScope

안드로이드에서는 스코프를 직접 정의하는 대신 다음 2가지를 사용할 수 있습니다.
- `viewModelScope`
- `lifecycleScope`

이들의 작동 방식은 `Dispatchers.Main`과 `SupervisorJob`을 사용하고, `ViewModel`이나 `LifecycleOwner`가 소멸될 때 코루틴의 `Job`을 취소합니다.

```kotlin
// Implementation from lifecycle-viewmodel-ktx version 2.4.0
public val ViewModel.viewModelScope: CoroutineScope 
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        
        if (scope != null) { 
            return scope
        } 
        
        return setTagIfAbsent(JOB_KEY, ClosableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate))
    }

internal class ClosableCoroutineScope(
    context: CoroutineContext
): Closeable, CoroutineScope {
    
    override val coroutineContext: CoroutineContext = context
    
    override fun close() { 
        coroutineContext.cancel()
    }
}
```

`viewModelScope`와 `lifecycleScope`는 스코프에 특별한 컨텍스트(`CoroutineExceptionHandler` 등)이 필요하지 않은 경우에 권장됩니다.

```kotlin
class ArticlesListViewModel(
    private val produceArticlesUseCase: ProduceArticlesUseCase
) : ViewModel() {
    
    private val _progressBarVisible = MutableStateFlow(false)
    val progressBarVisible: StateFlow<Boolean> = _progressBarVisible

    private val _articlesListState = MutableStateFlow<ArticlesListState>(Initial)
    val articlesListState: StateFlow<ArticlesListState> = _articlesListState

    fun onCreate() {
        viewModelScope.launch {
            _progressBarVisible.value = true
            val articles = produceArticlesUseCase.produce()
            _articlesListState.value = ArticlesLoaded(articles)
            _progressBarVisible.value = false
        }
    }
}
```