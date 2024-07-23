Compose에서 컴포저블은 실제로 UI를 생성하는 것이 아닌, Composer를 통해 런타임이 관리하는 인메모리 구조(슬롯 테이블)에 변경 사항을 "방출"함을 알아야 합니다.  
이와 같은 표현을 나중에 해석하여 UI를 "구체화" 해야 합니다.

이처럼, 컴포저블들이 컴포지션에 변경 사항을 "방출"하고, 방출된 변경 사항을 통해 컴포지션을 업데이트할 수 있습니다.  
이 업데이트는 컴파일러로 인해 주입된 $composer 인스턴스를 통해 이루어집니다.   
현재 Composer 인스턴스를 얻는 호출과 컴포지션 자체는 Compose 런타임의 일부입니다.

지금까지 런타임에 의해 메모리에 유지되는 상태를 “컴포지션”이라고 지칭했습니다. 이는 의도적으로 피상적인 개념입니다.  
컴포지션 상태를 저장하고 업데이트하는 데 사용되는 데이터 구조에 대해 알아보는 것부터 시작해 보겠습니다.

## The slot table and the list of changes

Compose에서 두 데이터 구조의 차이에 대한 이해가 필요합니다.

슬롯 테이블(Slot table)은 런타임이 **현재 컴포지션의 상태**를 저장하는데 사용하는 최적화된 인메모리 구조입니다.  
슬롯 테이블은 초기 컴포지션 동안 컴포저블의 호출과 관련된 모든 데이터를 저장하며 여기에는 다음과 같은 정보가 포함됩니다:

- 소스 코드 내 컴포저블의 위치
- 컴포저블에 전달된 파라미터
- `remember` 값
- `CompositionLocal`

그 다음 재구성이 일어날 때마다, 저장된 데이터가 업데이트 됩니다.  
슬롯 테이블에 저장된 정보들은 Composer가 다음 변경 사항 목록(list of changes)을 생성하는데 사용됩니다.  
이는 컴포지션 트리에 대한 모든 변경 사항은 현재 컴포지션의 상태에 따라 달라지기 때문입니다.

슬롯 테이블이 현재 컴포지션의 상태를 기록하는 반면, 변경 사항 목록(list of changes)은 노드 트리에 실제로 변경을 가하는 역할을 합니다.  
변경 사항 목록은 일종의 패치 파일로 이해할 수 있으며, 이를 적용하면 트리가 업데이트 됩니다.  
모든 변경 사항은 기록된 후에 적용되어야 합니다. 이 목록에 있는 변경 사항을 적용하는 것은 `Applier`의 역할입니다.    
`Appiler`는 런타임이 트리를 실제로 구성하는데 의존하는 추상화된 개념입니다.

마지막으로, `Recomposer`는 언제, 어떤 스레드에서 재구성을 수행할지, 그리고 언제, 어떤 스레드에서 변경 사항을 적용할지를 결정합니다.

## The slot table in depth

컴포지션의 상태가 어떻게 저장되는지 알아보겠습니다.

슬롯 테이블은 '빠른 선형 접근'에 최적화된 데이터 구조로, 텍스트 편집기에서 자주 사용되는 "gap buffer" 개념을 기반으로 합니다.  
"gap buffer" 개념을 기반으로 하기에, 다음과 같이 데이터를 두 개의 선형 배열에 저장합니다:

- 그룹 배열: 컴포지션에 있는 그룹에 대한 정보를 저장합니다.
- 슬롯 배열: 각 그룹에 속한 슬롯(slot)을 저장합니다.

> Chapter2에서 컴파일러가 컴포저블 함수의 본문을 래핑하여 그룹을 방출하는 방법에 대해 배웠습니다.  
> 이 그룹들을 메모리에 저장하면 컴포저블에 아이덴티티(유니크 키)를 부여하여 나중에 식별할 수 있게 합니다.  
> 그룹은 컴포저블 호출과 그 자식에 대한 정보를 래핑하고, 컴포저블을 그룹으로 처리하는 방법에 대한 정보를 제공합니다.  
> 그룹은 컴포저블 본문 내에서 발견되는 제어 흐름 패턴에 따라 다양한 유형을 가질 수 있습니다:  
> Restartable 그룹, Moveable 그룹, Replaceable 그룹, Reusable 그룹 등...

```kotlin
var groups = IntArray(0)
    private set

var slots = Array<Any?>(0) { null }
    private set
```

그룹 배열(`groups`)은 그룹에 대한 메타데이터를 나타내는 "그룹 필드"를 저장하기 위해 `Int` 값을 사용합니다.  
그룹 배열은 부모 그룹과 자식 그룹의 형태로 저장되고, 선형 데이터 구조이기에 부모 그룹이 먼저 오고, 그 다음에 모든 자식 그룹의 그룹 필드가 이어집니다.  
이는 그룹 트리를 선형 방식으로 모델링하는 방법으로, 자식들의 선형 검색을 용이하게 합니다.  
그룹 앵커를 통하지 않은 무작위 접근은 비용이 많이 드며, 앵커(`Anchors`)는 이 목적을 위해 존재하는 포인터와 같습니다.

| Group 1 (Parent) | Group 2 (Child) | Group 3 (Child) | Group 4 (Parent) | Group 5 (Child) | Group 6 (Child) |
|------------------|-----------------|-----------------|------------------|-----------------|-----------------|

한편, 슬롯 배열(`slots`)은 각 그룹에 대한 관련 데이터를 저장합니다.  
**슬롯 배열은 모든 유형의 값을 저장할 수 있도록 `Any?` 타입을 사용하며, 실제 컴포지션 데이터는 여기에 저장됩니다.**  
그룹은 항상 특정 슬롯 범위와 연결되어 있기 때문에, 그룹 배열에 저장된 각 그룹은 슬롯 배열에서 자신의 슬롯을 찾고 해석하는 방법을 설명합니다.

슬롯 테이블은 읽기와 쓰기를 위해 갭(gap)에 의존하며, 이를 테이블의 포지션 범위로 생각할 수 있습니다.  
이 갭은 배열에서 데이터의 읽기와 쓰기 위치를 결정하기 위해 이동할 수 있습니다.  
또한, 갭은 데이터 쓰기의 시작 위치를 가리키는 '포인터'를 가지며, 시작 위치와 끝 위치를 변경할 수 있어 테이블의 기존 데이터도 덮어쓸 수 있습니다.

![gap.png](gap.png)

다음 조건부 로직을 생각해보세요:

```kotlin
@Composable
@NonRestartableComposable
fun ConditionalText() {
    if (a) {
        Text(a)
    } else {
        Text(b)
    }
}
```

위와 같이 컴포저블이 `@NonRestartableComposable`로 처리된 경우, Restartable 그룹 대신, Replaceable 그룹이 삽입됩니다.  
Replaceable 그룹은 현재 "활성화된 자식"에 대한 데이터를 테이블에 저장합니다. 즉, `a`가 `true`인 경우, 이는 `Text(a)`가 됩니다.  
만약, 조건이 변경되면 갭은 그룹의 시작 위치로 다시 이동하고, 거기서부터 쓰기를 시작하여 `Text(b)`에 대한 데이터로 모든 슬롯을 덮어씁니다.

슬롯 테이블에서는 데이터의 '읽기/쓰기'를 `SlotReader`와 `SlotWriter`로 처리하며, 다음 특징을 가집니다:

- 각 '읽기/쓰기' 작업 후, 해당 Reader/Writer는 닫힙니다.
- 슬롯 테이블은 여러 개의 활성화된 Reader를 가질 수 있지만, 활성화된 Writer는 오직 하나만 가질 수 있습니다.
- 원하는 만큼 많은 Reader를 열 수 있지만, 안전을 위해 테이블이 쓰여지는 동안에는 읽을 수 없습니다.
- 활성화된 Writer는 그룹과 슬롯을 직접 수정하므로, 동시에 읽고 쓰기 작업을 하게되면 경쟁 조건이 발생되기에, 슬롯 테이블은 활성 Writer가 닫힐 때까지 유효하지 않습니다.

Reader는 방문자(visitor)로 동작하며, 현재 그룹 배열에서 다음을 추적합니다:

- 현재 읽고 있는 그룹
- 현재 읽고 있는 그룹의 시작과 끝 위치
- 바로 앞에 저장된 부모 그룹
- 현재 읽고 있는 그룹의 슬롯
- 그룹이 가진 슬롯의 개수

| Group 1 (Parent) | Group 2 (Child) | Group 3 (Child) | Group 4 (Parent) | Group 5 (Child) | Group 6 (Child) |
|------------------|-----------------|-----------------|------------------|-----------------|-----------------|

| 현재 읽고 있는 그룹 | 시작 위치 | 끝 위치 |  부모 그룹  | 그룹의 슬롯 인덱스 | 슬롯 개수 |
|:-----------:|:-----:|:----:|:-------:|:----------:|:-----:|
|   Group 1   |   0   |  3   |  NONE   |     0      |   3   |
|   Group 2   |   1   |  2   | Group 1 |     1      |   1   |
|   Group 3   |   2   |  3   | Group 1 |     2      |   1   |

Reader는 위치를 재조정하고, 그룹을 건너뛰고, 현재 슬롯에서 값을 읽고, 특정 인덱스의 값을 읽는 등의 작업을 수행할 수 있습니다.  
즉, Reader는 배열에서 그룹과 그 슬롯에 대한 정보를 읽기 위해 사용됩니다.

반면, Writer는 그룹 배열과 슬롯 배열에 쓰기 작업을 수행하기 위해 사용됩니다.  
앞서 설명한 것처럼, Writer는 `Any?` 타입의 데이터를 테이블에 쓸 수 있습니다.  
`SlotWriter`는 앞서 언급한 그룹과 슬롯의 갭을 사용하여 배열 내에서 데이터를 쓸 위치를 결정합니다.

선형 배열의 위치에 갭을 슬라이드하고 크기를 조절할 수 있는 범위로 생각해보면, Writer는 각 갭의 시작 위치, 끝 위치, 길이를 추적합니다.  
갭의 시작과 끝 위치를 업데이트하여 갭을 이동시킬 수 있습니다.

Writer는 그룹과 슬롯을 추가, 교체, 이동 및 제거할 수 있습니다.  
예를 들어, 트리에 새로운 컴포저블 노드를 추가하거나, 조건부 로직에 따라 컴포저블을 교체하는 것을 생각할 수 있습니다.

Writer는 그룹과 슬롯을 건너뛸 수 있고, 주어진 위치만큼 전진하거나, 앵커(`Anchor`)에 의해 결정된 위치로 이동하는 등 다양한 작업을 수행할 수 있습니다.

Writer는 특정 인덱스로 빠르게 접근하기 위해 앵커 목록을 유지합니다.  
테이블에서 각 그룹의 위치(인덱스)는 앵커를 통해 추적됩니다.  
앵커는 그룹이 이동, 교체, 삽입, 제거될 때 업데이트됩니다.

슬롯 테이블은 컴포지션 그룹의 반복자(Iterator)로도 작동하므로, 툴이 컴포지션의 세부 사항을 검사하고 정보를 제공할 수 있습니다.

## The list of changes

컴포지션(또는 재구성)이 발생할 때마다 소스의 Composable 함수가 실행되고 방출됩니다. "방출"이라는 단어를 이미 여러 번 사용했습니다.
방출은 슬롯 테이블을 업데이트하고, 궁극적으로 구체화된 트리도 업데이트하기 위한 지연된 변경 사항을 생성하는 것을 의미합니다.
이러한 변경 사항은 목록으로 저장됩니다. 이 새로운 변경 목록을 생성하는 것은 이미 슬롯 테이블에 저장된 내용을 기반으로 합니다.
기억할 점은 트리에 대한 모든 변경 사항은 컴포지션의 현재 상태에 따라 달라져야 한다는 것입니다.

> 예를 들어, 노드가 이동하는 경우를 생각해보면 리스트의 Composable 순서를 재정렬하는 경우,
> 해당 노드가 테이블에서 어디에 위치했는지 확인하고, 해당 슬롯을 모두 제거하고, 새 위치에서 다시 쓰는 작업이 필요합니다.

이 말은 Composable이 방출할 때마다 슬롯 테이블을 살펴보고, 필요에 따라 현재 정보를 사용하여 지연된 변경을 생성하고, 모든 변경 사항을 포함하는 목록에 추가한다는 것을 의미합니다.
나중에 컴포지션이 완료되면 구체화할 시간이 되어 기록된 변경 사항이 실제로 실행됩니다. 그떄 슬롯 테이블이 컴포지션의 최신 정보로 업데이트됩니다.
이 과정은 단순히 실행 대기 중인 지연된 작업을 생성하는 것이기에 방출 프로세스를 매우 빠르게 만듭니다.

이로 인해 변경 목록이 테이블에 변경 사항을 반영하는 역할을 합니다. 그 후, Applier에게 알려 물리적 노드 트리를 업데이트합니다.

위에서 언급한 것처럼 Recomposer는 이 과정을 조율하고 어떤 스레드에서 컴포지션 또는 재구성을 수행할지, 그리고 변경 목록에서 변경 사항을 적용할 스레드를 결정합니다.
후자는 또한 LaunchedEffect가 effect를 실행하는 데 사용하는 기본 컨텍스트가 될 것입니다.

이제 변경 사항이 기록되고 지연되며, 최종적으로 실행되는 방법과 모든 상태가 슬롯 테이블에 저장되는 방법에 대한 더 명확한 이해를 얻었습니다.

## The Composer

주입된 `$composer`는 작성한 Composable 함수를 Compose 런타임에 연결하는 역할을 합니다.

## Feeding the Composer

노드가 메모리 내 트리 표현에 어떻게 추가되는지 `Layout` Composable을 통해 살펴보겠습니다.
`Layout`은 Compose UI에서 제공하는 모든 UI 구성 요소의 배관 역할을 합니다. 코드에서는 다음과 같습니다.

```kotlin
@Suppress("ComposableLambdaParameterPosition")
@Composable
inline fun Layout(
    content: @Composable () -> Unit,
    modifier: Modifier = Modifier,
    measurePolicy: MeasurePolicy
) {
    val density = LocalDensity.current
    val layoutDirection = LocalLayoutDirection.current
    ReusableComposeNode<ComposeUiNode, Applier<Any>>(
        factory = ComposeUiNode.Constructor,
        update = {
            set(measurePolicy, ComposeUiNode.SetMeasurePolicy)
            set(density, ComposeUiNode.SetDensity)
            set(layoutDirection, ComposeUiNode.SetLayoutDirection)
        },
        skippableUpdate = materializerOf(modifier),
        content = content
    )
}
```

`Layout`은 `ReusableComposeNode`을 통해 컴포지션에 LayoutNode를 방출합니다.  
이 방법이 노드를 생성하고 즉시 추가하는 것처럼 보이지만, 실제로는 런타임에 노드를 생성, 초기화 및 삽입하는 방법을 알려주는 것입니다.

```kotlin
@Composable
inline fun <T, reified E : Applier<*>> ResuableComposeNode(
    noinline factory: () -> T,
    update: @DisallowComposableCalls Updater<T>.() -> Unit,
    noinline skippableUpdate: @Composable SkippableUpdater<T>.() -> Unit,
    content: @Composable () -> Unit
) {
    // ...
    currentComposer.startReusableNode()
    // ...
    currentComposer.createNode(factory)
    // ...
    Updater<T>(currentComposer).update() // initialization
    // ...
    currentComposer.startReplaceableGroup(0x7ab4aae9)
    content()
    currentComposer.endReplaceableGroup()
    currentComposer.endNode()
}
```

여기선 아직 중요하지 않은 부분을 생략했지만, 모든 작업을 `currentComposer` 인스턴스에 위임하는 것을 볼 수 있습니다.
또한 컴포지션에 저장할 때 이 Composable의 `content`를 래핑하기 위해 replaceable 그룹을 시작하는 것을 볼 수 있습니다.
`content` 람다 내에서 방출된 모든 자식은 이 그룹(따라서 Composable)의 자식으로 컴포지션에 저장됩니다.

다른 Composable 함수에도 동일한 방출 작업이 수행됩니다.   
예를 들어 `remember`를 살펴보겠습니다:

```kotlin
@Composable
inline fun <T> remember(calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(invalid = false, calculation)
```

`remember` 함수는 제고된 람다에 의해 반환된 값을 컴포지션에 캐싱하기 위해 `currentComposer`를 사용합니다.  
`invalid` 파라미터는 이전에 저장된 값과 관계없이 값을 업데이트하도록 강제합니다. `cache` 함수는 다음과 같이 작성됩니다:

```kotlin
@Composable
inline fun <T> Composer.cache(invalid: Boolean, block: () -> T): T {
    return rememberedValue().let {
        if (invalid || it === Composer.Empty) {
            val value = block()
            rememberedValue(value)
            value
        } else {
            it
        }
    } as T
}
```

먼저, 컴포지션(슬롯 테이블)에서 값을 찾습니다.  
값을 찾지 못하면 해당 값을 업데이트하도록 스케줄링하고, 그렇지 않으면 해당 값을 반환합니다.

## Modeling the Changes

모든 방출 작업은 `currentComposer`에 위임되며, 내부적으로는 변경 사항이으로 모델링되어 목록에 추가됩니다.  
변경 사항은 현재 `Applier`와 `SlotWriter`에 접근할 수 있는 지연 함수입니다. (한 번에 하나의 writer만 활성화됩니다.)

코드로 살펴보겠습니다:

```kotlin
internal typealias Change = (
    applier: Applier<*>,
    slots: SlotWriter,
    rememberManager: RememberManager
) -> Unit
```

"방출" 행위는 본질적으로 이러한 변경 사항을 생성하는 것을 의미하며, 이는 슬롯 테이블에서 노드를 추가, 제거, 교체 또는 이동하고 `Applier`에게 알리는 지연된 람다입니다.(따라서 이러한 변경 사항이
구체화될 수 있도록 합니다.)

따라서 "변경 사항을 방출한다"는 말을 할 때 "변경 사항을 기록한다" 또는 "변경 사항을 스케줄한다"라는 단어를 사용할 수 있습니다. 모두 같은 의미입니다.

컴포지션이 완료되고, 모든 Composable 함수 호출이 완료되고 모든 변경 사항이 기록되면, Applier에 의해 모든 변경 사항이 일괄적으로 적용됩니다.

> 컴포지션 자체는 `Composition` 클래스로 모델링됩니다.   
> 이는 이후 이 장의 뒷부분에서 자세히 다룰 예정이므로 지금은 제외합니다. 먼저 Composer에 대해 더 자세히 알아보겠습니다.