Jetpack Compose에 대해 이야기할 때, 일반적으로 컴파일러, 런타임, 그리고 경우에 따라 Compose UI까지 포함한 모든 구성 요소를 하나로 묶어 말합니다.
이전 섹션에서는 컴파일러에 대해 배우고, 그것이 런타임에서 최적화와 다양한 기능을 가능하게 하는 방법을 알아보았습니다.
그 다음으로, 런타임 자체를 공부하면서, Compose의 실제 메커니즘이 어떻게 작동하고, 그 모든 기능과 강력함이 어디에 있는지를 발견했습니다.
이제는 Compose UI, 즉 런타임을 사용하는 클라이언트 라이브러리를 살펴볼 차례입니다.

## Integrating UI with the Compose runtime

Compose UI는 Kotlin 멀티플랫폼 프레임워크로, 컴포저블을 사용해 UI를 생성할 수 있는 기본 구성 블록(building blocks)과 메커니즘을 제공합니다.
또한, 이 라이브러리에는 Android와 Desktop 플랫폼을 위한 통합 레이어를 제공하는 소스셋(source set)이 포함되어 있습니다.

> JetBrains는 Desktop 소스셋을 적극적으로 관리하며, Google은 Android와 공통 소스셋을 관리합니다.  
> Android와 Desktop 소스셋은 공통 소스셋에 의존합니다.   
> 한편, Compose for Web은 DOM을 사용하여 구축되기 때문에, 현재 Compose UI와는 별도로 유지되고 있습니다.

Compose 런타임과 UI 프레임워크를 통합할 때의 목표는 사용자가 화면에서 경험할 수 있는 '레이아웃 트리'를 구축하는 것입니다.  
이 트리는 UI를 생성하는 컴포저블을 실행하여 만들어지며, 이후에 필요에 따라 업데이트됩니다.  
이 트리에 사용되는 노드 타입은 Compose UI만이 알고 있으므로, 런타임은 이런 세부 사항에 대해 알 필요 없이 독립적으로 유지됩니다.

비록 Compose UI 자체가 이미 Kotlin 멀티플랫폼 프레임워크이지만, 현재 지원되는 노드 타입은 Android와 Desktop에 한정되어 있습니다.  
Compose for Web과 같은 다른 라이브러리는 다른 노드 타입을 사용합니다.   
이러한 제한으로 인해, 클라이언트 라이브러리가 생성하는 노드 타입은 해당 클라이언트 라이브러리에서만 알아야 합니다.  
이 때문에 런타임은 트리에서 노드를 삽입-제거-이동-교체하는 작업을 클라이언트 라이브러리에 위임합니다.

초기 컴포지션 과정과 이후의 재구성 과정은 레이아웃 트리를 구축하고 업데이트하는데 중요한 역할을 합니다.  
이 과정에서는 컴포저블을 실행하여, 트리에서 노드를 삽입-제거-이동-교체하는 '변경 사항'을 예약하게 됩니다.  
이 변경 사항들은 이후 `Applier`를 통해 트리의 구조에 영향을 미치는 변경 사항을 감지하고, 이를 실제 트리 변경으로 반영하여 사용자가 경험할 수 있도록 합니다.

초기 컴포지션에서는 이 변경 사항들이 모든 노드를 삽입하여 레이아웃 트리를 구축하며, 재구성에서는 트리를 업데이트합니다.  
재구성은 컴포저블의 입력 데이터(i.e : 파라미터 또는 읽어들인 가변 상태)가 변경될 때 트리거됩니다.

## Mapping scheduled changes to actual changes to the tree

컴포지션 또는 재구성 과정에서 컴포저블이 실행될 때, 변경 사항을 발행합니다.  
이 과정에서 `Composition`이라는 사이드 테이블(side table)이 사용됩니다.

이 사이드 테이블은 컴포저블의 실행으로 발행한 예약된 변경 사항(scheduled changes)을 노드 트리의 실제 변경 사항(actual changes)으로 매핑하는데 필요한 데이터를 포함하고 있습니다.

Compose UI를 사용하는 애플리케이션에서는 표현해야 하는 노드 트리의 수만큼 많은 `Composition`을 가질 수 있습니다.  
지금까지는 `Composition`이 여러 개일 수 있다는 언급이 없었기에 다소 놀라울 수 있습니다. 그러나, 실제로 `Composition`은 여러 개일 수 있습니다.

이제부터 이에 대해 더 자세히 알아보면서, 레이아웃 트리가 어떻게 구축되는지, 어떤 노드 타입이 사용되는지 빠르게 이해할 것입니다.

## Composition from the point of view of Compose UI

Android로 예시를 들면, Compose UI 라이브러리에서 런타임으로 가장 자주 진입하는 지점은 특정 화면을 구성하기 위한 `setContent`를 호출할 때입니다.

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            MaterialTheme {
                Text("Hello, World!")
            }
        }
    }
}
```

그러나 화면(e.g : Activity/Fragment)만이 `setContent` 호출이 발생할 수 있는 유일한 곳은 아닙니다.  
예를 들어, 하이브리드 Android 앱에서 `ComposeView`를 통해 뷰 계층 구조의 중간에서도 `setContent`를 호출할 수 있습니다:

```kotlin
ComposeView(requireContext()).apply {
    setContent {
        MaterialTheme {
            Text("Hello, World!")
        }
    }
}
```

이 예제에서는 뷰를 프로그래밍 방식으로 생성하고 있지만, 이 뷰는 앱에서 XML로 정의된 레이아웃 계층의 일부로도 포함될 수 있습니다.

`setContent`는 새로운 루트 `Composition`을 생성하고, 가능한 경우 이를 재사용합니다.  
이를 "루트" 컴포지션으로 부르는 이유는 각각이 독립적인 컴포저블 트리를 호스팅하기 때문입니다.  
이 컴포지션들은 서로 연결되지 않으며, 각 컴포지션은 표현하는 UI의 복잡성에 따라 단순하거나 복잡할 수 있습니다.

이러한 개념을 바탕으로, 앱 내에 여러 개의 노드 트리가 있고, 각각이 다른 `Composition`과 연결될 수 있다고 상상해 볼 수 있습니다.

예를 들어, 3개의 `Fragment`가 있는 Android 앱을 생각해보면(아래 그림 참조), Fragment 1과 3은 `setContent`를 호출하여 각자의 컴포저블 트리를 연결합니다.
반면, Fragment 2는 레이아웃에서 여러 개의 `ComposeView`를 선언하며 각각의 `setContent`를 호출합니다.
이 경우, 앱에는 5개의 '루트 컴포지션'이 존재하게 되며, 이들은 모두 완전히 독립적으로 작동합니다.

![multiple root composition.png](multiple_root_composition.png)

이러한 레이아웃 계층 중 어느 하나를 생성하기 위해, 해당하는 Composer는 컴포지션 과정을 실행합니다.  
이 과정에서 해당 `setcontent` 호출에 포함된 모든 컴포저블들이 실행되어 변경 사항을 발행합니다.

Compose UI에서는 이러한 변경 사항이 UI 노드의 삽입-이동-교체를 포함하며, 이는 주로 `Box`, `Column`, `LazyColumn`과 같은 일반적인 UI 구성 블록들에 의해 이루어집니다.
이러한 컴포저블들이 서로 다른 라이브러리(`foundation`, `material`)에 속하더라도, 모두 최종적으로는 `Layouts`(`compose-ui`)로 정의되며, 동일한 노드 타입인 `LayoutNode`를 내보내게 됩니다.

`LayoutNode`는 UI 블록을 나타내기에, Compose UI에서 루트 컴포지션으로 가장 자주 사용되는 노드 타입입니다.

모든 `Layout` 컴포저블은 컴포지션에 `LayoutNode`를 발행하며, 이는 `ReusableComposeNode`를 통해 이루어집니다.
(`ComposeUiNode`는 `LayoutNode`에 의해 구현된 계약(contract)임에 유의하세요):

```kotlin
@Composable
inline fun Layout(
    measurePolicy: MeasurePolicy,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    val density = LocalDensity.current
    val layoutDirection = LocalLayoutDirection.current
    val viewConfiguration = LocalViewConfiguration.current

    // Emits a LayoutNode
    ReusableComposeNode<ComposeUiNode, Applier<Any>>(
        factory = { LayoutNode() },
        update = {
            set(measurePolicy, { this.measurePolicy = it })
            set(density, { this.density = it })
            set(layoutDirection, { this.layoutDirection = it })
            set(viewConfiguration, { this.viewConfiguration = it })
        },
        skippableUpdate = materializerOf(modifier),
        content = content
    )
}
```

위 코드는 'ReusableNode'를 컴포지션에 삽입하거나 업데이트하는 변경 사항을 발행합니다.  
이러한 과정은 개발자들이 사용하는 모든 UI 구성 블록에서 동일하게 이루어집니다.

> ReusableNode는 Compose 런타임에서의 최적화 기법입니다.  
> 노드의 키가 변경될 때, ReusableNode를 사용하면 Composer가 노드를 삭제하고 새로 만드는 대신, 그 자리에서 노드의 `content`를 재구성(update)할 수 있습니다.
> 이를 위해, 컴포지션은 새로운 `content`를 생성하는 것처럼 동작하지만, 슬롯 테이블은 재구성이 일어나는 것처럼 탐색됩니다.
> 
> 이러한 최적화는 `set` 및 `update` 연산 정보만으로 완전히 정의될 수 있는 노드, 즉 내부에 숨겨진 상태가 없는 노드에만 가능합니다.
> 예를 들어, `LayoutNode`는 이러한 조건을 충족하지만, `AndroidView`는 그렇지 않기 때문에 ReusableNode 대신 표준 `ComposeNode`를 사용합니다.

`ReusableComposeNode`는 노드를 생성하고(팩토리 함수를 통해), 이를 초기화한 다음(`update` 람다로), 노드의 모든 `content`를 래핑하는 Replaceable 그룹을 생성합니다.
이 그룹에는 나중에 식별할 수 있도록 유니크 키가 할당됩니다.   
Replaceable 그룹 내에서 `content` 람다가 호출되어 발행된 모든 노드는 이 노드의 자식이 됩니다.

`update` 블록 내의 `set` 호출들은 노드가 처음 생성되었을 때나, 해당 속성의 값이 마지막으로 기억된 이후 변경된 경우에만, 후행 람다를 실행하도록 예약됩니다.

이렇게 해서 `LayoutNode`가 애플리케이션에 존재하는 각각의 여러 컴포지션에 전달됩니다.  
이로 인해 모든 컴포지션이 `LayoutNode`만을 포함한다고 생각할 수 있지만, 이는 잘못된 생각입니다.  
Compose UI의 런타임에 전달되는 요소들을 이해할 때, 고려해야 할 다른 타입의 컴포지션과 노드들도 있습니다.