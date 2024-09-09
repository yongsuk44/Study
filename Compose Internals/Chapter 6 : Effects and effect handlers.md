이펙트 핸들러를 다루기 전에, 사이드 이펙트(side effect)가 무엇인지 다시 한 번 살펴보는 것이 좋습니다.  
사이드 이펙트를 이해하면, 왜 컴포저블 트리에서 사이드 이펙트를 제어하는 것이 중요한지에 대한 맥락을 이해할 수 있습니다.

## Introducing side effects

사이드 이펙트는 Chapter 1에서 컴포저블의 속성을 배울 때 다루었습니다.  
사이드 이펙트는 함수의 비결정성(non-deterministic)을 초래하여, 개발자 코드를 이해하고 추론하기 어렵게 만드는 것을 배웠습니다.

본질적으로 사이드 이펙트는 함수의 제어 및 스코프를 벗어나는 모든 것을 의미합니다.  
두 숫자를 더하는 함수를 예로 들어보겠습니다:

```kotlin
// Add.kt
fun add(a: Int, b: Int): Int = a + b
```

위 함수는 입력값만을 사용해 결과를 계산하기에 "순수" 함수라고도 불립니다.  
이 함수는 입력값이 동일하면 항상 같은 결과를 반환하므로, 결정론적(determinisitic)이라고 할 수 있으며, 쉽게 이해할 수 있습니다.

이제, 여기에 부수적인 동작을 추가해 보겠습니다:

```kotlin
// AddWithSideEffect.kt
fun addWithSideEffect(a: Int, b: Int): Int =
    calculateionsCache.get(a, b)
        ?: (a + b).also { calculationsCache.store(a, b, it) }
```

위 코드를 보면, '계산 캐시'를 추가하여 이미 계산된 결과가 있을 경우 계산 시간을 절약합니다.  
그러나, 이 캐시는 함수의 제어를 벗어나므로, 마지막 실행 이후 값이 변경되지 않았을 거라는 보장이 없습니다.

만약, 다른 스레드에서 캐시가 동시에 업데이트 된다면, 동일한 입력 값으로 `get(a, b)`를 두 번 연속 호출했을 때, 서로 다른 결과가 반횐될 수 있습니다:

```kotlin
// AddWithSideEffect2.kt
fun main() {
    add(1, 2) // 3
    // Another thread calls: cache.store(1, 2, res = 4)
    add(1, 2) // 4
}
```

`add` 함수가 동일한 입력값에 대해 서로 다른 값을 반환하기에, 더 이상 결정론적이지 않습니다.  
이와 마찬가지로, 캐시가 메모리가 아닌 데이터베이스에 의존한다고 가정해보겠습니다.  
데이터베이스 연결이 끊겼다면, `get` 및 `store` 호출에서 예외가 발생할 수 있으며, 이런 예상치 못한 시나리오는 `add` 호출도 실패할 수 있습니다.

요약하자면 사이드 이펙트는 함수 호출자가 기대하는 것과 다른 예상치 못한 동작을 의미하며, 이로 인해 함수의 동작이 변경될 수 있습니다.  
사이드 이펙트는 개발자가 코드를 추론하기 어렵게 만들며, 테스트 가능성을 제거하여 불안정성을 초래할 수 있습니다.

사이드 이펙트는 다음과 같은 다양한 예시들이 있습니다.

- 전역 변수를 읽고 쓰는 것
- 메모리 캐시에 접근하는 것
- 데이터베이스를 사용하는 것
- 네트워크 요청을 보내는 것
- 화면에 무언가를 출력하는 것
- 파일에서 데이터를 읽는 것

## Side effects in Compose

컴포저블 내에서 사이드 이펙트가 실행되면, 동일한 문제가 발생할 수 있다는 점을 배웠습니다.  
이렇게 되면, 사이드 이펙트가 컴포저블의 라이프사이클에서 벗어나게 되어, 제어와 제약을 잃게 됩니다.

또한, 모든 컴포저블이 여러 번 리컴포지션될 수 있다는 점도 배웠습니다. 그렇기에, 컴포저블 내에서 직접적으로 사이드 이펙트를 실행하는 것은 권장되지 않습니다.
이 내용은 Chapter 1에서 컴포저블은 재시작 가능(restartable)하다고 언급했습니다.

컴포저블 내부에서 사이드 이펙트를 실행하는 것은 코드의 무결성과 애플리케이션 상태를 손상시킬 수 있기에 위험합니다.  
Chapter 1에서 언급한, 네트워크에서 상태를 로드하는 컴포저블 예시를 다시 살펴보겠습니다:

```kotlin
@Composable
fun EventsFeed(
    networkService: EventsNetworkService
) {
    val events = networkService.loadAllEvents() // Side effect
    
    LazyColumn {
        items(events) { event ->
            Text(text = event.name)
        }
    }
}
```

이 예시에서 매번 리컴포지션될 때마다 사이드 이펙트가 실행되며, 이는 의도한 동작과는 다를 가능성이 높습니다.  
런타임은 짧은 시간 안에 이 컴포저블을 여러 번 리컴포지션 할 수 있는데, 그 결과로 많은 사이드 이펙트가 동시다발적으로 실행되면서, 서로 조율되지 않는 문제가 발생할 수 있습니다.
의도한 동작은 첫 번째 컴포지션에서만 사이드 이펙트를 실행하고, 이후 컴포저블 라이프사이클 동안 상태를 유지하는 것이었을 것입니다.

사용 사례를 Android UI라고 가정하면, `compose-ui`를 사용하여 컴포저블 트리를 빌드하며, 모든 Android 애플리케이션에는 사이드 이펙트가 존재합니다.  
외부 상태를 업데이트하는 사이드 이펙트의 예시를 살펴보겠습니다.

```kotlin
@Composable
fun MyScreen(
    drawerTouchHandler: TouchHandler
) {
    val drawerState = rememberDrawerState(DrawerValue.Closed)
    drawerTouchHandler.enabled = drawerState.isOpen
    
    // ...
}
```

이 컴포저블은 '터치 핸들링을 지원하는 드로어'가 포함된 화면을 나타냅니다.  
드로어 상태는 `Closed`로 초기화되지만, 시간이 지남에 따라 `Open`으로 변경될 수 있습니다.  
컴포지션 및 리컴포지션이 발생할 때마다, 컴포저블은 현재 드로어 상태를 `TouchHandler`에 알려, 드로어가 `Open` 상태일 때만 터치 핸들링이 활성화되도록 합니다.

`drawerTouchHandler.enabled = drawerState.isOpen` 라인은 사이드 이펙트입니다.  
이는 **컴포지션의 사이드 이펙트**로 외부 객체인 `TouchHandler`에 콜백 참조를 할당하고 있습니다.

이미 설명했듯이, 컴포저블 본문에서 사이드 이펙트는 언제 실행할 지 제어할 수 없다는 문제가 있습니다.  
그 결과, 사이드 이펙트는 컴포지션 및 리컴포지션할 때마다 실행되고, 또한 **절대 폐기되지 않아** 잠재적인 메모리 누수가 발생할 수 있습니다.

네트워크 요청을 트리거하는 컴포저블 예시로 다시 돌아가 보겠습니다.  
만약, 사이드 이펙트로 네트워크 요청을 실행한 컴포저블이, 완료되기 전에 컴포지션에서 떠난다면 어떤 일이 발생할까요?  
아마 개발자들은 그 시점에서 작업을 취소하는 것이 더 적절할 수 있습니다.

사이드 이펙트는 상태를 가지는 프로그램에서 필수적이기 떄문에, Compose는 라이프사이클을 고려한 방식으로 사이드 이펙트를 실행할 수 있는 메커니즘을 제공합니다.
이를 통해 작업을 리컴포지션 동안 유지하거나, 컴포저블이 컴포지션에서 벗어날 때 자동으로 작업을 취소되도록 할 수 있습니다. 
이러한 메커니즘을 **이펙트 핸들러**(effect handler)라고 합니다.

## What we need

컴포지션은 **다른 스레드에서 오프로드**되어 병렬로 실행될 수 있으며, 실행 순서가 달라질 수 있으며, 그 외 다양한 런타임 실행 전략이 있습니다.  
이는 Compose 팀이 다양한 최적화 가능성을 열어 두기 위한 전략이며, 따라서 아무런 제어 없이 컴포지션 중에 이펙트를 바로 실행하는 것은 바람직하지 않습니다. 

전반적으로 다음을 보장하는 메커니즘이 필요합니다:

- 컴포저블 라이프사이클에서 적절한 시점에 이펙트가 실행되도록 해야 합니다. 너무 이르거나 늦지 않게, 컴포저블이 준비되었을 때 실행되어야 합니다.
- 일시 중단된 이펙트는 적절히 구성된 런타임(코루틴과 적절한 `CoroutineContext`)에서 실행될 수 있어야 합니다.
- 참조를 캡처한 이펙트는 컴포지션을 벗어날 때, 이를 정리할 수 있는 기회를 가져야 합니다.
- 진행 중인 일시 중단된 이펙트는 컴포지션을 벗어날 때 취소되어야 합니다.
- 시간이 지나며 변하는 입력에 의존하는 이펙트는 입력이 변할 때마다 자동으로 정리/취소되고 다시 실행되어야 합니다.

이러한 메커니즘은 Jetpack Compose에서 제공되며, **이펙트 핸들러**(Effect handlers)라고 불립니다.

## Effect Handlers

이펙트 핸들러를 설명하기 전에, `@Composable` 라이프사이클을 간단히 살펴보겠습니다.

컴포저블은 화면에 표시될 때 컴포지션에 진입하고, UI 트리에서 제거되면 컴포지션을 떠납니다.  
이 두 이벤트 사이에서 이펙트가 실행될 수 있습니다. 일부 이펙트는 컴포저블의 라이프사이클보다 오래 지속되어 컴포지션을 넘어 계속 실행될 수 있습니다. 

지금은 이 정도 기본 개념을 알아두면 충분합니다. 이어서 진행하겠습니다.

이펙트 핸들러는 두 가지 카테고리로 나눌 수 있습니다:

- 비중단 이펙트(Non suspended effects) : 예를 들어, 컴포저블이 컴포지션에 진입할 때 콜백을 초기화하고, 떠날 때 이를 정리하는 이펙트.
- 중단 이펙트(Suspended effects) : 예를 들어, 네트워크에서 데이터를 로드하여 UI 상태에 데이터를 공급하는 이펙트.

## Non suspended effects

### DisposableEffect

`DisposableEffect`는 컴포지션 라이프사이클과 관련된 사이드 이펙트를 나타냅니다.

- 비중단 이펙트 중에서도 폐기(dispose)가 필요한 경우에 사용됩니다.
- 컴포저블이 컴포지션에 들어갈 때 처음 실행되고, 이후 키 값이 변경될 때마다 다시 실행됩니다.
- 마지막에 `onDispose` 콜백이 필요합니다. 컴포저블이 컴포지션에서 떠날 때 or 키 값이 변경될 때마다 폐기되며, 이 경우 이펙트가 폐기되고 다시 실행됩니다.

```kotlin
// DisposableEffect.kt
@Composable
fun backPressHandler(
    onBackPressed: () -> Unit,
    enabled: Boolean = true,
) {
    val dispatcher = LocalOnBackPressedDispatcherOwner.current.onBackPressedDispatcher
    
    val backCallback = remember {
        object : OnBackPressedCallback(enabled) {
            override fun handleOnBackPressed() {
                onBackPressed()
            }
        }
    }
    
    DisposableEffect(dispatcher) {      // dispose/relaunch if dispatcher changes
        dispatcher.addCallback(backCallback)
        
        onDispose { 
            backCallback.remove()       // avoid leaks
        }
    }
}
```

위 예제는 `CompositionLocal`에서 얻은 디스패처에 콜백을 연결하는 백프레스 핸들러가 있습니다.  
컴포저블이 컴포지션에 들어갈 때와 디스패처가 변경될 때 콜백을 연결하려면, **디스패처를 이펙트 핸들러의 키로 전달**하면 됩니다.   
이렇게 하면 디스패처가 변경될 때, 이펙트가 자동으로 폐기되고 다시 실행됩니다.

컴포저블이 컴포지션에서 완전히 떠날 떄, 콜백도 함께 폐기됩니다.

컴포지션에 진입할 때 한 번만 실행하고, 컴포지션을 떠날 때 폐기하려면, **상수를 키로 전달**하면 됩니다: `DisposableEffect(true)` or `DisposableEffect(Unit)`. 

`DisposableEffect`는 항상 최소한 하나의 키를 요구한다는 점을 기억해야 합니다. 

### SideEffect

컴포지션의 또 다른 사이드 이펙트입니다. 이 이펙트는 조금 특별한데, "이 컴포지션에서 실행하거나 잊어버리기"와 같은 방식입니다.  
만약 컴포지션이 어떤 이유로 실패한다면, 이펙트는 **폐기**됩니다.

Compose 런타임에 익숙하다면, 이 이펙트가 **슬롯 테이블에 저장되지 않는다**는 점을 기억하세요.  
이는 컴포지션이 종료되면 이펙트가 유지되지 않으며, 이후 컴포지션에서 다시 시도되거나 실행되지 않습니다.

- **폐기할 필요가 없는** 이펙트에 적합합니다.
- 매 컴포지션 / 리컴포지션 후에 실행됩니다.
- **외부 상태에 대한 업데이트를 게시**하는 데 유용합니다.

```kotlin
// SideEffect.kt
@Composable
fun MyScreen(
    drawerTouchHandler: TouchHandler
) {
    val drawerState = rememberDrawerState(DrawerValue.Closed)
    
    SideEffect {
        drawerTouchHandler.enabled = drawerState.isOpen
    }
    
    // ...
}
```

위 예시는 드로어의 현재 상태를 중요하게 생각하며, 이 상태는 언제든지 변경될 수 있습니다. 이런 의미에서, 매번 컴포지션이나 리컴포지션이 발생할 때마다 상태를 알릴 필요가 있습니다. 
또한, `TouchHandler`가 애플리케이션 실행 동안 항상 살아있는 싱글톤으로 메인 화면(항상 보이는 화면)에 사용된다면, 참조를 폐기하지 않는 것이 더 좋을 수 있습니다. 

`SideEffect`는 Compose `State` 시스템에서 관리하지 않는 외부 상태에 **업데이트를 게시**하여, 항상 동기화되도록 유지하는 이펙트 핸들러로 이해할 수 있습니다.

### currentRecomposeScope

`currentRecomposeScope`는 이펙트 핸들러보단 이펙트 자체에 더 가깝지만, 다뤄볼 가치가 있는 흥미로운 주제입니다.

Android 개발자라면 View 시스템의 `invalidate`와 비슷한 개념에 익숙할 수 있습니다.  
`invalidate`는 뷰에서 새로운 측정, 레이아웃, 드로우 단계를 강제로 실행하는 메서드입니다.  
예를 들어, `Canvas`를 사용한 프레임 기반 애니메이션을 만들 때 자주 사용됩니다.  
매번의 드로우 틱마다 뷰를 무효화하여, 경과된 시간에 따라 다시 드로우를 수행하는 방식입니다.

`currentRecomposeScope`는 다음과 같은 단일 목적을 가진 인터페이스입니다:

```kotlin
// RecomposeScope.kt
interface RecomposeScope {
    /**
     * Invalidate the corresponding scope, requesting the composer recompose this scope.
     */
    fun invalidate()
}
```

`currentRecomposeScope.invalidate()`를 호출하면 컴포지션을 로컬에서 무효화하고, **리컴포지션을 강제**할 수 있습니다.

이 방식은 **Compose의 상태 스냅샷이 아닌**, 다른 데이터의 출처(source of truth)를 사용할 때 유용할 수 있습니다.

```kotlin
interface Presenter {
    fun loadUser(after: @Composable () -> Unit): User
}

@Composable
fun MyComposable(presenter: Presenter) {
    val user = presenter.loadUser { currentRecomposeScope.invalidate() } // Not a State
    
    Text(text = "The loaded user: ${user.name}")
}
```

위 코드를 보면, 프레젠터를 사용해 결과가 있을 때, `invalidate`를 호출하여 수동으로 리컴포지션을 강제하고 있습니다.  
이는 `State`를 전혀 사용하지 않기 때문에 발생하는 상황으로, 매우 특이한 상황입니다.  
대부분의 경우에는 `State`와 스마트 리컴포지션을 사용하는 것이 좋습니다.

전반적으로, 이 방식은 신중하게 사용해야 합니다. 가능한 한 `State`를 사용해 스마트 리컴포지션을 활용하면 Compose 런타임의 성능을 최대한 활용할 수 있습니다.

> 프레임 기반 애니메이션을 위해 Compose는 choreographer에서 다음 렌더링 프레임까지 대기할 수 있는 API를 제공합니다.  
> 이후 실행이 재개되면, 경과된 시간 등을 사용하여 상태를 업데이트하고, 이를 통해 스마트 리컴포지션을 다시 활용할 수 있습니다.

## Suspended effects

### rememberCoroutineScope

`rememberCoroutineScope`는 컴포지션의 자식으로 간주될 수 있는 작업을 생성하는 `CoroutineScope`를 생성합니다.

- 컴포지션 라이프사이클에 바인딩된 중단 이펙트를 실행하는데 사용됩니다.
- 컴포지션 라이프사이클에 바인딩된 `CoroutineScope`를 생성합니다.
- 컴포지션을 떠날 떄 스코프가 취소됩니다.
- 컴포지션 간에 동일한 스코프가 반환되므로, 여러 작업을 계속해서 제출할 수 있으며, 컴포지션이 종료될 때 진행 중인 모든 작업이 취소됩니다.
- 사용자 상호작용에 대한 응답으로 작업을 시작하는데 유용합니다.
- 컴포지션에 진입할 때 이펙트는 applier 디스패처(일반적으로 `AndroidUiDispatcher.Main`)에서 실행됩니다.

```kotlin
// rememberCoroutineScope.kt
@Composable
fun SearchScreen() {
    val scope = rememberCoroutineScope()
    var currentJob by remember { mutableStateOf<Job?>(null) }
    var items by remember { mutableStateOf<List<Item>>(emptyList()) }
    
    Column {
        Row {
            TextField(
                value = "Start typing to search",
                onValueChange = { text -> 
                    currentJob?.cancel()
                    currentJob = scope.async {
                        delay(threshold)
                        items = viewModel.search(query = text)
                    }
                }
            )
        }
        Row { ItemsVerticalList(items) }
    }
}
```

이는 UI에서 쓰로틀링을 구현하는 방법입니다. 과거에 `View` 시스템에서 `postDelayed` or `Handler`를 통해 비슷한 작업을 했을 수 있습니다.  
텍스트 입력이 변경될 때마다 진행 중인 작업을 취소하고, 새 작업을 실행하기 전에 지연 시간을 두어, 네트워크 요청과 같은 작업 사이에 최소한의 지연 시간을 보장합니다. 

`LaunchedEffect`와의 차이점은 `LaunchedEffect`는 컴포지션에서 시작된 작업을 스코핑하는데 사용되는 반면, `rememberCoroutineScope`는 **사용자 상호작용으로 시작된 작업**을 스코핑하는데 사용된다는 것입니다.

### LaunchedEffect

`LaunchedEffect`는 컴포저블이 컴포지션에 진입하자마자 초기 상태를 로드하는 일시 중단 함수입니다.

- 컴포지션에 진입할 때 이펙트를 실행합니다.
- 컴포지션을 떠날 때 이펙트를 취소합니다.
- 키가 변경될 때 이펙트를 취소하고 다시 실행합니다.
- 리컴포지션 동안 작업을 지속하는데 유용합니다.
- 컴포지션에 진입할 때 이펙트는 applier 디스패처(일반적으로 `AndroidUiDispatcher.Main`)에서 실행됩니다.

```kotlin
// LaunchedEffect.kt
fun SpeakerList(eventId: String) {
    var speakers by remember { mutableStateOf<List<Speaker>>(emptyList()) }
    
    LaunchedEffect(eventId) {   // cancelled / relaunched when eventId varies
        speakers = viewModel.loadSpeakers(eventId)  // suspended effect
    }
    
    ItemsVerticalList(speakers)
}
```

설명할게 많지 않습니다. 
이 이펙트는 컴포지션에 진입할 떄 한 번 실행되고, 키 값에 의존하기 떄문에 키 값이 변경될 때마다 다시 실행됩니다. 컴포지션을 떠날 때 이펙트가 취소됩니다.

또한, 다시 실행되어야 할 때마다 이펙트가 취소된다는 점을 기억해야 합니다.  
`LaunchedEffect`는 최소한 하나의 키가 필요합니다.