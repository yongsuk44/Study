## The nature of Composable functions

Composable 함수는 Compose의 기본 빌딩 블록이자, 'Composable 트리'를 작성하는 데 사용되는 구조입니다. 
트리로 표현하는 이유는, Compose 런타임이 Composable 함수를 메모리에 표현할 수 있는 큰 트리의 노드로 나타낼 수 있기 때문입니다.

기본 문법으로는 Kotlin 함수에 `@Composable` 어노테이션을 추가하는 것만으로 Composable 함수를 만들 수 있습니다.

```kotlin
@Composable
fun NamePlate(name: String) { ... }
```

위와 같은 방식으로 컴파일러에게 `NamePlate`의 데이터를 Composable 트리의 노드로 변환할 것임을 알릴 수 있습니다.  
즉, `@Composable (Input) -> Unit`의 'Input'은 데이터이고, `Unit`은 함수 반환 값이 아니라 요소를 메모리 내의 Composable 트리에 등록하는 작업을 의미합니다.

> 'Input'을 받아들이는 함수가 'Unit'을 반환하는 것은 함수 본문 내에서 'Input'을 어떻게든 소비하고 있다는 의미입니다.

이러한 작업은 Compose 용어로 'emitting'이라고 하며, 이는 노드 트리에 변경 사항을 추가하는 것을 의미합니다.  
Composable 함수는 실행될 때 'emitting' 작업을 수행하며, 이는 Composition 단계에서 이루어집니다.

그러나, 모든 Composable 함수가 `Unit`을 반환하는 것은 아닙니다.
일부 함수는 값을 반환하며, 이는 Composable 함수의 의미를 변경합니다.
이러한 함수들은 'Input'을 "소비"하는 것이 아니라, 'Input'을 기반으로 값을 "제공"하는 것으로 이해할 수 있습니다.  

여기에는 `remember`가 대표적인 예시입니다.

```kotlin
@Composable
fun NamePlate() {
    val name = remember { generateName() }
    Text(name)
}
```

`remember` Composable 함수는 연산 결과를 기억해두고, 다음 번에 같은 연산을 할 때 이전 결과를 바로 반환합니다. 이를 메모이제이션(memoization)이라고 합니다.
`remember`가 실행될 때마다 Compose는 UI 트리의 일관성을 유지하기 위해 이 함수의 호출 정보와 결과를 메모리에 저장합니다. 
따라서, `remember`는 단순히 결과를 기억하는 것뿐만 아니라 관련된 모든 정보를 저장하여 UI 트리가 항상 최신 상태를 유지하도록 돕습니다.

`@Composable` 어노테이션은 해당 함수에게 몇 가지 중요한 속성을 갖게 합니다. 이 속성은 **Compose 런타임이 다양한 최적화를 수행**할 수 있게합니다. 
예를 들어, 병렬 Composition, 우선순위에 따른 임의의 Composition, 스마트 재구성, 위치 기반 메모이제이션 등의 최적화가 가능해집니다.

런타임 최적화는 런타임이 실행해야 할 코드에 대해 특정 조건과 동작을 가정할 수 있을 때 가능합니다.  
예를 들어, 코드 내의 요소들이 서로 의존하지 않는다면 병렬로 실행할 수 있고, 순서를 바꿔도 결과에 영향을 주지 않는다면 임의의 순서로 실행할 수 있습니다.

Composable 함수가 이러한 최적화가 가능한 이유는 함수들이 서로 독립적이며, 동일한 입력에 대해 항상 동일한 결과를 반환하기 때문입니다. 
이러한 특성은 Compose 런타임이 효율적인 최적화를 수행할 수 있도록 합니다.

## Calling context

Composable 함수는 Compose 컴파일러에 의해 암묵적으로 Composer 컨텍스트 인스턴스를 파라미터로 받는 함수로 변환됩니다.
이 인스턴스는 해당 함수의 Composable 자식들에게도 전달됩니다. 이는 런타임과 개발자가 이를 명시적으로 다룰 필요 없이, 암시적으로 주입된 파라미터로 생각하면 됩니다.

예를 들어, 다음 Composable 함수가 있다고 가정해봅시다:

```kotlin
@Composable
fun NamePlate(name: String, lastname: String) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text(text = name)
        Text(
          text = lastname,
          style = MaterialTheme.typography.body2
        )
    }
}
```

컴파일러는 각 Composable 호출에 암묵적인 Composable 파라미터를 추가하고, 각 Composable의 시작과 끝에 몇 가지 마커를 추가합니다.
다음 코드는 단순화된 예시이지만, 결과는 다음과 같을 것입니다:

```kotlin
fun NamePlate(name: String, lastname: String, $composer: Composer) {
    $composer.start(123)
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text(
          text = name,
          $composer
        )
        Text(
          text = lastname,
          style = MaterialTheme.typography.body2,
          $composer
        )
    }
    $composer.end()
}
```

이렇게 하면 Composer 컨텍스트가 트리를 따라 전달되어, 트리가 Composable 함수로만 구성된 경우 어느 레벨에서든 항상 사용할 수 있습니다.
컴파일러는 Composable 함수가 Composable 함수에서만 호출될 수 있도록 엄격한 규칙을 적용합니다.
다시 말해, Composable 함수를 호출하기 위해서는 호출 컨텍스트(calling context)가 필요합니다.

이 요구 사항을 통해 Compose는 런타임에 필요한 정보가 어떤 서브트리에서도 항상 접근 가능하도록 합니다. 
Composable 함수는 실제 UI를 생성하는 대신 트리에 변경 사항을 'emitting'(발행) 합니다.
Composable 함수는 Composition 중에 이러한 변경 사항을 발행하기 위해 주입된 Composer 인스턴스를 사용하며, 이후의 재구성은 이전 실행에서 발행된 변경 사항에 따라 달라집니다.
이것이 작성한 코드와 Compose 런타임을 연결하는 방식입니다. 이 연결을 통해 런타임은 트리의 구조를 파악하고, 메모리 내에서 이를 표현하여 다양한 최적화를 수행할 수 있습니다.

## Idempotent

Composable 함수가 갖는 또 다른 특성은 프로그램 상태에 대해 'idempotent(멱등성)'을 유지해야 한다는 것입니다.  
이는 Composable 함수에 동일한 데이터를 입력하면, 항상 동일한 프로그램 상태를 반환해야 한다는 것을 의미합니다.

Compose 런타임은 재구성과 같은 작업에서 이 가정을 사용합니다.

> Compose에서 재구성은 의존하는 데이터가 변할 때 Composable 함수를 다시 호출하여 업데이트된 요소를 발행하는 작업을 의미합니다.
> 이를 UI로 번역하면, 함수는 UI 트리에 업데이트되거나 새로운 노드를 발행할 수 있습니다.

Compose에서 재구성은 여러 가지 이유로 발생할 수 있으며, 함수는 여러 번 재구성될 수 있습니다.  
그렇기 때문에 Composable 함수가 멱등성을 유지하는 것은 매우 중요합니다. 그렇지 않으면 재구성될 때마다 프로그램 상태가 부작용으로 인해 변경될 수 있습니다.

재구성은 트리를 따라 내려가면서 어떤 노드가 재구성되어야 하는지 확인하는 수직적인 작업입니다.  
"스마트 재구성"이란 변경된 데이터에 의존하지 않은 Composable 트리의 부분을 재구성하지 않고 그대로 유지하는 것을 의미합니다.  
이는 효율성 면에서 큰 발전이며, Composable 람다가 다시 호출되지 않기 때문에 가능합니다.
한 번 더 생각해보면, 이는 Composable 함수가 이미 메모리에 저장되어 있고, 입력이 변하지 않을 때 그대로 사용할 수 있기 때문입니다.
이것이 바로 멱등성의 직접적인 결과입니다.

만약, Composable 함수가 동일한 입력에 대해 항상 동일한 프로그램 상태를 반환하지 않는다면, 런타임은 이러한 가정을 할 수 없으며 최적화를 수행할 수 없습니다.

## Free of side effects

"순수 함수" 또는 "순수성"이라는 용어는 부작용(side effects)가 없는 함수를 의미합니다.

> 여기서 "부작용"이란 함수의 범위를 벗어나 예상치 못한 작업을 수행하는 모든 동작을 의미합니다.
> Compose 관점에서는, 부작용이란 Composable 함수의 범위를 벗어나 프로그램 상태를 변경하는 동작을 의미합니다.
> 더 넓게 보면, 전역 변수를 설정하거나, 메모리 캐시를 업데이트하거나, 네트워크 쿼리를 수행하는 것도 부작용으로 간주될 수 있습니다.
> 네트워크 쿼리가 실패하면 어떻게 될까요? 또는 함수 실행 사이에 외부 캐시가 업데이트되면 어떻게 될까요? 함수의 동작은 이러한 요소에 의존합니다.
> 부작용은 함수의 비결정론적 동작을 초래하여 프로그램 상태의 일관성을 해칩니다. 또한 부작용은 프로그램에서 경쟁 조건(race conditions)을 초래할 수 있습니다.

Composable 함수가 멱등성을 유지하려면 제어되지 않은 부작용을 실행하지 않아야 한다는 점을 짐작할 수 있습니다. 
그렇지 않으면 함수는 다시 실행될 때마다 다른 프로그램 상태를 반환할 수 있으며, 멱등성을 유지하지 못하게 됩니다.
따라서, 이 두 속성은 실제로 매우 밀접하게 연결되어 있습니다.

부작용을 허용하면 Composable 함수가 이전 Composable 함수의 실행 결과에 의존하게 될 수 있습니다.
이는 절대로 허용해서는 안됩니다. 왜냐하면 런타임은 Composable 함수가 어떤 순서로든, 심지어 병렬로 실행될 수 있음을 기대하기 때문입니다.
예를 들어, 재구성을 다른 스레드로 분산시키고 여러 코어를 활용할 수 있게 합니다.

```kotlin
@Composable
fun MainScreen() {
    Header()
    ProfileDetail()
    EventList()
}
```

위 예시에서 Header, ProfileDetail, EventList Composable은 어떤 순서로든 실행될 수 있으므로, 순차적으로 실행된다고 가정해서는 안됩니다.

이런 점에서, 순서에 의존하는 로직을 작성하지 않아야 합니다. 
예를 들어, `ProfileDetail`에서 상태를 업데이트하여 `EventList`에서 새로운 효과를 트리거하려고 하는 경우를 생각해보면, 
`EventList`가 `ProfileDetail`보다 먼저 실행되거나 동시에 실행되는 경우는 어떻게 될까요?
또 다른 예로는 `Header`에서 전역 변수를 설정하고 `EventList`에서 이를 읽는 경우를 들 수 있습니다.
만약 순서를 바꾼다면 프로그램은 어떻게 동작해야 할까요? 
Composable 사이에서 부작용을 통해 관계를 설정하는 것은 대부분 잘못된 것이며 피해야 합니다.
비즈니스 로직을 작성해야 하는 경우, 이는 Composable 함수 하나 또는 여러 개의 책임이 아니며, 다른 아키텍처 계층에 위임해야 합니다.

> Compose는 우선순위에 따라 Composition을 재정렬할 수 있는 능력을 가지고 있습니다.  
> 예를 들어, 화면에 표시되지 않는 Composable에 낮은 우선순위를 할당할 수 있습니다.

Composable 함수에서 직접 부작용을 실행하면, 재구성의 부작용으로 인해 여러 번 호출될 수 있습니다.
이는 코드와 애플리케이션 상태의 무결성을 저해하고, 경쟁 조건을 초래할 수 있어 매우 위험합니다.

예를 들어, Composable 함수에서 네트워크를 통해 데이터를 로드하는 경우를 생각해보겠습니다:

```kotlin
@Composable
fun EventFeed(service: NetworkService) {
    val events = service.loadEvents()
    
    LazyColumn {
        items(events) { event ->
            EventCard(event)
        }
    }
}
```

여기서 부작용은 매 재구성 마다 다시 발생하며, 매우 짧은 시간 안에 `EventFeed`이 여러 번 재구성될 경우, 여러 부작용이 동시에 발생할 수 있습니다.

하지만 상태를 유지하는 프로그램을 작성하려면 부작용이 필요하기 때문에, Compose는 Composable 함수 내에서 부작용을 안전하게 호출하는 메커니즘을 제공합니다.
이 메커니즘은 Composable 생명주기를 인식하도록 만들어져 있어, 재구성 동안 작업을 분산시킬 수 있게 합니다. 
이를 'effect handlers'라고 부릅니다. 지금은 Composable 본문에서 직접 부작용을 호출하지 않도록 안전한 환경을 제공한다고 이해하면 됩니다.

제어 없이 Composable에서 부작용을 실행할 때 발생할 수 있는 또 다른 문제는 외부 변수를 업데이트하는 경우입니다.
Composable 함수는 다양한 스레드에서 실행될 수 있으므로, 해당 변수에 대한 접근은 'thread-safe' 하지 않습니다.

```kotlin
@Composable
fun BuggyEventFeed(events: List<Event>) {
    var totalEvents = 0
    
    Column {
        Text(
            if (totalEvents == 0) "No events" 
            else "Total events: $totalEvents"
        )
        
        events.forEach { event ->
            Text(event.name)
            totalEvents++
        }
    }
}
```

여기서 `totalEvents`는 Column Composable이 재구성될 때마다 수정됩니다. 이는 총 개수가 일치하지 않고, 경쟁 조건에 노출됩니다.

> 마지막으로 부작용이 없는 코드에 중점을 두는 이유는 사용자가 작성하는 코드가 예측 가능하고 일관되게 동작하도록 하기 위함입니다.
> 그러나, 실제로 Composition을 빌드하거나 업데이트하는 것은 Composable 호출 그래프를 실행함으로써 이루어지며, 이 자체가 사실상 부작용입니다.
> 하지만 이 부작용은 Compose 라이브러리의 기능을 활성화하고 프로그램 동작에 일관성을 해치지 않기 때문에 구조적으로 허용됩니다.

## Restartable

Composable 함수는 재시작 가능해야 합니다. 이는 이전 섹션에서 설명한 것과 같은 개념입니다.
Composable 함수는 재구성될 수 있으며, 이는 일반적인 함수와 달리 한 번만 호출되지 않는다는 것을 의미합니다. 
Composable 함수를 재구성한다는 것은 함수를 다시 실행하는 것을 의미합니다.

Compose는 Composable 함수가 언제든지, 필요할 때만 재구성할 수 있는 기능을 제공합니다.
이를 통해 호출 그래프의 어떤 노드를 재구성할지 선택적으로 결정할 수 있으며, 필요한 부분만 다시 시작할 수 있습니다.
이는 함수들이 관찰하는 상태의 변경에 따라 다시 실행될 수 있는 반응형 접근 방식입니다.

> 모든 Composable 함수는 기본적으로 재시작 가능하도록 설계되어 있습니다.
> 이는 Compose 런타임이 기대하는 바이지만, Compose 런타임은 `@NonRestartableComposable`이라는 어노테이션을 제공하여, 이 기능을 특정 Composable 함수에서 제거할 수 있습니다.
> 
> `@NonRestartableComposable`을 사용하면 컴파일러가 Composable 함수를 재구성하거나 건너뛰는데 필요한 보일러플레이트 코드를 생성하지 않습니다.
> 하지만, 아주 작은 함수들이 다른 Composable 함수에 의해 재구성될 가능성이 높은 경우에만 의미가 있을 수 있기 때문에 이 어노테이션을 사용할 때는 매우 신중해야 합니다.
> 작은 함수들은 매우 적은 논리를 포함하고 있어 자체적으로 무효화할 필요가 없기 때문에, 부모 또는 상위 Composable에 의해 무효화 및 재구성될 가능성이 높습니다.

## Fast execution

Composable 함수는 여러 번 호출될 수 있기에 빠르게 실행되어야 합니다.  
따라서 실제 UI를 즉시 생성하는 대신, Composable 트리에 노드 변경 사항을 발행하도록 설계되었습니다.

예를 들어, Composable 함수는 애니메이션의 매 프레임마다 호출될 수 있습니다. 
비용이 많이 드는 연산은 Coroutine으로 분산하고, 항상 생명주기를 인식하는 'effect-handler' 중 하나로 래핑해야 합니다.
Compose 컴파일러는 부작용을 생명주기에 맞추고 중단함으로써 이를 올바르게 사용하고 Compose 런타임이 기대하는 대로 조정할 수 있도록 합니다.

Composable 함수와 Composable 함수 트리는 메모리에 유지되어 추후에 해석/구체화될 프로그램의 설명을 작성하는 빠르고 선언적이며 가벼운 접근 방식으로 생각할 수 있습니다.