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
Composable 함수는 실행될 때 'emitting' 작업을 수행하며, 이는 컴포지션 단계에서 이루어집니다.

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
예를 들어, 병렬 컴포지션, 우선순위에 따른 임의의 컴포지션, 스마트 재구성, 위치 기반 메모이제이션 등의 최적화가 가능해집니다.

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
Composable 함수는 컴포지션 중에 이러한 변경 사항을 발행하기 위해 주입된 Composer 인스턴스를 사용하며, 이후의 재구성은 이전 실행에서 발행된 변경 사항에 따라 달라집니다.
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

> Compose는 우선순위에 따라 컴포지션을 재정렬할 수 있는 능력을 가지고 있습니다.  
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
> 그러나, 실제로 컴포지션을 빌드하거나 업데이트하는 것은 Composable 호출 그래프를 실행함으로써 이루어지며, 이 자체가 사실상 부작용입니다.
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

## Positional memoization

이 속성을 이해하기 전에, "함수 메모이제이션(function memoization)"에 대해 먼저 알아야 합니다.
함수 메모이제이션은 함수가 입력에 따라 결과를 캐시할 수 있는 능력을 의미합니다. 따라서 함수가 동일한 입력에 대해 호출될 때마다 다시 계산할 필요가 없습니다.
위에서 설명한 것처럼, 이는 순수(결정론적) 함수에만 가능합니다. 왜냐하면 동일한 입력에 대해 항상 동일한 결과를 반환할 것이라는 확신이 있기 때문에 결과를 저장하고 재사용할 수 있습니다.

> 함수 메모이제이션은 함수형 프로그래밍 세계에서 널리 알려진 기술로, 프로그램이 순수 함수의 조합으로 정의되기에 함수의 결과를 메모이제이션하여 성능을 크게 향상시킬 수 있습니다.

위치 기반 메모이제이션(Positional Memoization)은 이 아이디어를 기반으로 하지만, 중요한 차이점이 있습니다.
Composable 함수는 Composable 트리에서 자신의 위치를 항상 알고 있습니다. 
런타임은 동일한 Composable 함수에 대한 호출을 부모 내에서 고유한 아이덴티티를 제공하여 구분합니다.
이 아이덴티티는 Composable 함수 호출의 위치를 기반으로 생성됩니다. 이를 통해 런타임은 다음과 같은 코드에서 Text() Composable 함수에 대한 세 가지 호출을 구분할 수 있습니다:

```kotlin
@Composable
fun MyComposable(txt: String) {
    Text(txt)
    Text(txt)
    Text(txt)
}
```

이 세가지 호출은 동일한 Text Composable에 대한 호출이며, 동일한 입력을 가지고 있습니다.
그러나, 동일한 부모 내에서 다른 위치에서 호출되기에 컴포지션은 각각 다른 아이덴티티를 가진 세 개의 인스턴스를 생성합니다.

이 아이덴티티는 재구성 동안에도 유지되므로, 런타임은 컴포지션을 통해 특정 Composable이 이전에 호출되었는지, 또는 변경되었는지를 확인할 수 있습니다.
때때로 이러한 아이덴티티를 생성하는 것은 런타임에게 어려울 수 있습니다. 왜냐하면 소스 코드 내에서 호출 위치에 의존하기 때문입니다.

예를 들어, 루프에서 생성된 Composable 목록의 경우 호출 위치가 여러 호출에 대해 동일할 수 있지만, 여전히 다른 노드로 인식되어야 합니다.

```kotlin
@Composable
fun TalksScreen(talks: List<Talk>) {
    Column {
        for (talk in talks) {
            Talk(talk)
        }
    }
}
```

여기서 `Talk(talk)`는 매번 동일한 위치에서 호출되지만, 각 `Talk`는 다를 것으로 기대합니다.
위와 같은 경우에 런타임은 호출 순서에 의존하여 유니크 아이디를 생성하고 이를 구분할 수 있습니다.
이는 목록 끝에 새 요소를 추가할 때 잘 작동하지만, 목록의 맨 위나 중간에 요소를 추가할 때에는 문제가 발생할 수 있습니다.
이 경우에는 해당 지점 아래의 모든 `Talk`를 재구성하게 되며, 이는 비효율적이고 예상치 못한 문제를 초래할 수 있습니다.

이러한 경우, 런타임은 `key` Composable를 통해 호출에 유니크 키를 수동으로 할당할 수 있도록 합니다.

```kotlin
@Composable
fun TalksScreen(talks: List<Talk>) {
    Column {
        for (talk in talks) {
            key(talk.id) { // Unique Key
                Talk(talk)
            }
        }
    }
}
```

이렇게 하면 각 `Talk(talk)` 호출을 `talk.id`에 기반하여 유니크하게 만들 수 있습니다. 
이를 통해 `Talk` 호출에 대한 아이덴티티를 보장할 수 있으며, 이는 목록의 순서가 변경되더라도 모든 항목의 아이덴티티를 유지할 수 있습니다.

Composable 함수가 "컴포지션에 노드를 발행한다."고 말할 때, 
Composable 호출과 관련된 모든 정보(파라미터, 내부 Composable 호출, remember 호출 결과 등)가 저장됩니다.
이는 주입된 Composer 인스턴스를 통해 이루어집니다.

Given Composable functions know about their location, any value cached by those will be cached only in the context delimited by that location. Here is an example for more clarity:

Composable 함수가 자신의 위치를 알고 있기에, 해당 위치에서 캐시된 값은 해당 위치에서만 캐시됩니다. 
이를 더 명확하게 설명하기 위해 예시를 들어보겠습니다:

```kotlin
@Composable
fun FilteredImage(path: String) {
    val filters = remember { computeFilters(path) }
    ImageWithFiltersApplied(filters)
}

@Composable
fun ImageWithFiltersApplied(filters: Filters) {
    // Apply filters to the image
}
```

여기서 `remember`를 사용하여 주어진 `path`의 이미지에 대한 필터를 미리 계산하여 무거운 연산의 결과를 캐싱합니다.
필터를 계산한 후, 이미지를 이미 계산된 피렅로 렌더링합니다. 연산 결과를 캐싱하는 것은 바람직합니다.
캐싱된 값을 인덱싱하는 키는 호출 위치와 함수 입력(`path`)을 기반으로 합니다.

> `remember`는 슬롯 테이블에서 결과를 읽는 방법을 알고 있는 Composable 함수입니다.
> 함수가 호출되면 테이블에서 함수 호출을 찾아 캐시된 결과를 반환합니다.
> 만약, 결과가 없다면 계산하고 반환하기 전에 결과를 저장합니다. 이렇게 하면 나중에 검색할 수 있습니다.

Compose에서 메모이제이션은 전통적인 "애플리케이션 전체" 메모이제이션과는 다릅니다.
여기서 `remember`는 위치 기반 메모이제이션을 사용하여 Composable 호출 범위 내에서 캐시된 값을 가져옵니다.
이는 해당 범위 내에서 단일 인스턴스처럼 작동합니다. 즉, 초기 컴포지션 동안에만 값을 계산하고, 재구성에서는 캐시된 값을 반환합니다.
하지만 동일한 Composable이 다른 컴포지션에서 사용되거나 다른 Composable에서 동일한 함수 호출이 기억되면, 기억된 값은 다른 인스턴스가 됩니다.

Compose는 위치 기반 메모이제이션 개념을 기반으로 구축되었으며, 스마트 재구성은 이를 기반으로 합니다.
`remember` Composable 함수는 명시적으로 더 세밀한 제어를 위해 이를 사용합니다.

## Similarities with suspend functions

Composable 함수가 호출 컨텍스트를 요구한다는 점은 Kotlin의 또 다른 언어 기본 기능인 suspend 함수와 유사합니다.

Kotlin에 익숙하면 suspend 함수 또한 호출 컨텍스트을 요구한다는 것을 알 것입니다.
즉, suspend 함수는 다른 suspend 함수에서만 호출될 수 있습니다. Kotlin 컴파일러는 이 규칙을 적용하여 suspend 함수 체인의 모든 함수를 새로운 버전으로 대체하고, 
이 버전은 각 단계에 암묵적인 파라미터를 전달합니다. 이 파라미터는 `Continuation`입니다.

```kotlin
suspend fun publishTweet(tweet: Tweet): Post = ...
```

위 코드를 Kotlin 컴파일러는 아래와 같이 변환합니다.

```kotlin
fun publishTweet(tweet: Tweet, continuation: Continuation<Post>): Unit
```

이때 Continuation은 컴파일러에 의해 추가되고 모든 suspend 호출에 전달됩니다.
이는 Kotlin 런타임이 다양한 suspend 지점에서 실행을 중단하고 재개하는데 필요한 모든 정보를 포함합니다.

suspend는 호출 컨텍스트를 강제하여 실행 트리를 통해 암묵적인 정보를 전달할 수 있는 방법을 보여주는 좋은 예입니다.

이런 의미에서, 우리는 @Composable을 언어 기능의 한 예로 이해할 수 있습니다.

> 여기서 생길 수 있는 합리적인 질문은 "Compose 팀은 왜 suspend를 사용하지 않았는가?" 입니다.
> 두 기능이 구현하는 패턴은 유사하지만, 언어에서는 완전히 다른 동작을 활성화합니다.
> 
> Continuation 인터페이스는 실행을 중단하고 재개하는 문제를 해결하는데 특화되어 있습니다.
> 이는 콜백 인터페이스로 모델링되며, Kotlin은 이를 위해 필요한 모든 메커니즘을 갖춘 기본 구현을 생성합니다.
> 반면, Compoes의 목표는 런타임에서 다양한 방식으로 최적화할 수 있는 큰 호출 그래프의 메모리 내 표현을 만드는 것입니다.
> 
> Compose 동작에서 필요한 점은 호출 컨텍스트를 강제하는 것뿐이며, 흐름 제어를 인코딩하는 것이 아닙니다.
> 흐름 제어 인코딩은 궁극적으로 Continuation이 하는 일입니다. 

## Composable functions are colored

Composable 함수는 일반 함수와 다른 기능과 속성을 가지고 있습니다. 이 함수들은 다른 타입을 가지며, 매우 특정한 관심사를 모델링합니다.
이러한 차별화는 일종의 "함수 컬러링"으로 이해할 수 있는데, 이는 별개의 함수 카테고리를 나타내기 떄문입니다.

함수 컬러링은 구글의 Dart 팀에서 작성한 블로그 포스트(What color is your function?)에서 설명한 개념입니다.
여기서는 async 함수와 sync 함수가 함께 잘 조합되지 않는다는 것을 설명하면서, 
sync 함수에서 async 함수를 호출하려면 sync 함수를 async로 만들거나 async 함수를 투명하게 호출하고 결과를 기다릴 수 있는 메커니즘을 제공해야 한다고 설명합니다.
그래서 일부 라이브러리와 언어에서는 조합성을 되찾으려 시도하기 위해 Promise와 async/await가 도입되었습니다.

Kotlin에서는 suspend가 이 문제를 어느 정도 해결하지만, 채색(colored)되어 있습니다.
suspend 함수는 특정한 실행 문맥(Continuation)이 필요하기에, 오직 suspend 함수에서만 suspend 함수를 호출할 수 있습니다.

이 문제의 해결 여부와 상관 없이, 이것이 일어나는 이유는 두 가지 다른 카테고리의 함수을 모델링하고 있기 때문입니다.
이는 두 개의 다른 언어를 사용하는 것과 같으며, 대부분의 경우 이들을 결합하는 것은 좋지 않은 결과를 초래합니다.
즉, 즉시 결과를 계산하는 연산(sync)과 시간이 걸리는 연산(async)이 있습니다.
후자는 파일 읽기, 데이터베이스 쓰기 또는 네트워크 서비스 쿼리와 같은 시간이 걸리는 작업에 주로 사용됩니다.

Composable 함수들은 불변 데이터를 트리의 노드에 매핑하는 재시작 가능하고 메모이제이션 가능한 함수를 나타냅니다.
Compoes 런타임은 이 사실에 의존합니다. 이것이 컴파일러가 Composable 함수가 다른 Composable 함수에서만 호출되도록 강제하는 이유입니다.
이러한 제한은 컴포지션을 활성화하고 구동하는 호출 문맥을 보장하기 위해 사용됩니다.

하지만, "함수 컬러링"의 근본적인 문제는 채색(colored)이 서로 다른 함수를 투명하게 조합할 수 없다는 점에서 비롯됩니다.
Compose에서는 Composable 함수는 Non-composable 함수에서 호출될 것으로 기대되지 않는데, 이는 Compose를 사용하는 방식이 Composable 함수들로만 그래프를 구축하기 때문입니다.
이들은 개발자가 트리를 구축하는데 사용하는 DSL의 원자 단위입니다. Composable 함수는 프로그램 로직을 작성하거나 일반 함수와 조합하는 용도로 사용되지 않습니다.

여기서 약간 속임수가 존재함을 알 수 있습니다. Composable 함수의 이점 중 하나는 로직을 통해 UI를 선언할 수 있다는 점입니다.
이는 때때로 일반 함수에서 Composable 함수를 호출해야 할 수 있음을 의미합니다.

예를 들어:

```kotlin
@Composable
fun SpeakerList(speakers: List<Speaker>) {
    Column {
        speakers.forEach {
            Speaker(it)
        }
    }
}

@Composable
fun Speaker(speaker: Speaker) {
    Text(speaker.name)
}
```

여기서 우리는 `forEach` 람다에서 Speaker Composable을 호출하고 있으며, 컴파일러는 이에 대해 불평하지 않습니다.

이처럼 함수 색상을 섞어 사용할 수 이유는 `inline`에 있습니다.
컬렉션 연산은 모두 `inline`으로 플래그가 지정되어 있어, 이들은 본질적으로 자신의 람다를 호출자에게 inline합니다.
위 예제에서 Speaker Composable 호출은 SpeakerList 본문 내에서 inline되며, 이는 둘 다 Composable 함수이기 때문에 허용됩니다.

따라서 API에서 inline을 활용함으로써 Composable 함수의 로직을 작성할 때 함수 색 구분 문제를 피할 수 있습니다.
Composable 함수 자체는 여전히 호출 문맥을 강제함으로써 색이 구분됩니다.
또한, Composable에서 기대되는 로직 타입은 보통 조건부 로직으로, 이는 특정 조건(보통 어떤 상태나 파라미터)에 따라 Composable을 교체하는 로직입니다.
이러한 로직은 대부분 단순하며, inline으로 작성되거나 필요한 경우 inline 함수로 분리할 수 있습니다.

Composable 함수가 색을 가지고 있다는 것을 이해하면, 이들이 반드시 다른 카테고리의 함수가 되어야함을 이해할 수 있습니다.
이를 통해 Compose 컴파일러가 이들을 다르게 처리하고, 이들의 동작과 호출 문맥에 대한 가정을 통해 이들의 능력을 활용할 수 있습니다.

## Composable function types

`@Composable` 어노테이션은 함수의 타입을 컴파일 시에 변경합니다. 더 구체적으로, Composable 함수는 `@Composable (T) -> Unit` 함수 타입을 따릅니다. 
여기서 `T`는 입력 데이터이고 `Unit`은 함수가 입력을 받아 트리에 변화를 주어야 함을 나타냅니다. 
개발자는 이 타입을 사용하여 표준 람다처럼 Composable 람다를 선언할 수 있습니다.

```kotlin
// This can be reused from any Composable tree
val textComposable: @Composable (String) -> Unit = { 
    Text(
        text = it,
        style = MaterialTheme.typography.body1
    )
}

@Composable
fun NamePlate(name: String, lastName: String) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text(
            text = name,
            style = MaterialTheme.typography.h6
        )
        textComposable(lastName)
    }
}
```

Composable 함수는 또한 수신자(receiver)를 가지는 람다가 필요할 때 `@Composable Scope.() -> Unit` 타입을 따를 수 있습니다.
이는 Composable DSL을 활용하여 Composable을 중첩할 수 있도록 하는 스타일로 자주 사용됩니다.
이러한 경우, 스코프는 일반적으로 modifier나 해당 Composable 내에서 읽을 수 있는 관련 변수를 포함합니다.

Composable 함수는 수신자(receiver)를 갖는 람다가 필요할 때 `@Composable Scope.() -> Unit` 타입을 따를 수도 있습니다. 
이는 Composable DSL을 사용하여 Composable 함수를 중첩할 수 있게 합니다. 
이러한 경우, 스코프는 보통 modifier나 해당 Composable 내에서 사용할 변수를 포함합니다.

```kotlin
inline fun Box(
    ...,
    content: @Composable BoxScope.() -> Unit
) {
    // ...
    Layout(
        content = { BoxScopeInstance.content() },
        measurePolicy = measurePolicy,
        modifier = modifier
    )
}
```

또한, Compose 컴파일러는 Composable 함수에 특정한 제한 사항과 속성을 부과합니다. 
이러한 요구 사항은 Composable 함수의 타입 정의의 일부로 볼 수 있습니다. 
타입은 데이터를 세분화하고 속성 및 제한 사항을 부여하여 컴파일러가 이를 정적으로 검사할 수 있게 합니다. 
이는 @Composable 어노테이션이 하는 일입니다.

Composable 함수의 요구 사항은 코드를 작성하는 동안에도 Compose 컴파일러에 의해 검사됩니다. 
컴파일러는 호출, 타입 및 선언 검사기를 사용하여 필요한 호출 문맥을 보장하고, 멱등성을 유지하며, 통제되지 않은 부작용을 방지합니다. 
이러한 검사기는 Kotlin 컴파일러의 프론트엔드 단계에서 실행되며, 이 단계는 정적 분석에 사용되며 가장 빠른 피드백을 제공합니다. 
이 라이브러리는 의도적으로 설계된 공개 API와 정적 검사를 통해 개발자에게 안내된 경험을 제공합니다.