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

초기 컴포지션 프로세스와 이후의 재구성 과정은 레이아웃 트리를 구축하고 업데이트하는데 중요한 역할을 합니다.  
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

이러한 레이아웃 계층 중 어느 하나를 생성하기 위해, 해당하는 Composer는 컴포지션 프로세스를 실행합니다.  
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

## Subcomposition from the point of view of Compose UI

컴포지션은 루트 레벨에만 존재하지 않고, 더 깊은 레벨의 컴포저블 트리에서도 생성될 수 있으며, 이때 부모 컴포지션과 연결됩니다.
Compose에서는 이를 '**서브 컴포지션**'이라고 부릅니다.

이전에 배운 것처럼, 컴포지션들은 트리 구조로 연결될 수 있습니다.  
각 컴포지션은 부모 컴포지션을 나타내는 `CompositionContext`에 대한 참조를 가지며, 이 `CompositionContext`는 해당 컴포지션의 부모 컴포지션을 의미합니다. (루트 컴포지션의 경우, 부모는 `Recomposer` 자체입니다)

이러한 구조 덕분에 런타임은 `CompositionLocal`과 무효화를 트리 전체에 걸쳐 단일 컴포지션처럼 처리하고 전파할 수 있습니다.

Compose UI에서 서브 컴포지션을 생성하는 중요한 두 가지 이유가 있습니다:

1. 일부 정보가 알려질 때까지 초기 컴포지션 프로세스를 연기하기 위함.
2. 서브 트리가 생성하는 노드의 타입을 변경하기 위함.

이 두 가지 이유에 대한 내용을 다루어 보겠습니다.

### 1. Deferring initial composition process

`SubcomposeLayout`에서 초기 컴포지션 프로세스를 연기하는 예시를 볼 수 있습니다.

`SubcomposeLayout`은 '레이아웃 단계'에서 독립적인 컴포지션을 생성하고 실행하는 `Layout`과 유사한 기능을 제공합니다.  
이를 통해 자식 컴포저블들이 해당 레이아웃에서 계산된 값에 의존할 수 있습니다.

예를 들어, `SubcomposeLayout`은 `BoxWithConstraints`에서 사용되며, 이 블록은 부모의 제약 조건(constraints)을 노출하여 콘텐츠를 그에 맞게 조정할 수 있습니다.
아래는 공식 문서에서 가져온 예제로, `BoxWithConstraints`를 사용하여 `maxHeight`에 따라 두 컴포저블 중 하나를 선택합니다.

```kotlin
BoxWithConstraints {
    val rectangleHeight = 100.dp
    
    if (maxHeight < rectangleHeight * 2) {
        Box(Modifier.size(50.dp, rectangleHeight).background(Color.Blue))
    } else {
        Column {
            Box(Modifier.size(50.dp, rectangleHeight).background(Color.Blue))
            Box(Modifier.size(50.dp, rectangleHeight).background(Color.Gray))
        }
    }
}
```

서브 컴포지션을 생성하는 주체는, 초기 컴포지션 프로세스의 실행 시점을 제어할 수 있습니다.   
즉, `SubcomposeLayout`은 초기 컴포지션 프로세스를 루트 컴포지션이 이루어질 때가 아닌, 레이아웃 단계에서 실행하도록 결정할 수 있습니다.

또한, 서브 컴포지션은 부모 컴포지션과 독립적으로 재구성을 수행할 수 있습니다.  
예를 들어, `SubcomposeLayout`에서 '레이아웃'이 발생할 때마다 람다에 전달되는 파라미터가 달라질 수 있으며, 이 경우에 서브 컴포지션에 대한 재구성이 트리거됩니다. 
그러나, 서브 컴포지션 내부에서 사용되는 상태가 변경될 경우, 초기 컴포지션이 완료된 후, 부모 컴포지션에서 재구성이 예약됩니다.

노드를 발행하는 관점에서 보면, `SubcomposeLayout`는 `LayoutNode`를 내보내므로, 서브 트리에서 사용되는 노드 타입과 부모 컴포지션에서 사용되는 노드 타입이 동일합니다.
여기에서 다음과 같은 질문이 생길 수 있습니다.

- Q : 단일 컴포지션 내에서 서로 다른 노드 타입을 지원하는 것이 가능할까?  
- A : 기술적으로는 가능하지만, 해당 `Applier`가 이를 허용하는 경우에만 가능합니다.  

결국에는 노드 타입이 무엇을 의미하는지에 따라 달라집니다.

만약 사용되는 노드 타입이 여러 서브 타입의 공통 부모라면, 서로 다른 노드 타입을 지원할 수 있습니다.  
하지만, 이 경우 `Applier`의 로직이 더 복잡해질 수 있습니다.  
실제로 Compose UI에서 사용 가능한 `Applier` 구현은 단일 노드 타입으로 고정되어 있습니다.

그럼에도 불구하고, 서브 컴포지션을 사용하면 서브 트리에서 완전히 다른 노드 타입을 지원할 수 있습니다.  
이는 앞서 언급한 서브 컴포지션의 두 번째 사용 사례입니다.

### Changing the node type in a subtree

Compose UI에는 서브 트리에서 노드 타입을 변경하는 좋은 예시가 있습니다.  
바로 벡터 그래픽을 생성하고 표시하는 컴포저블인 `rememberVectorPainter` 입니다.

벡터 컴포저블은 벡터 그래픽을 트리 구조로 모델링하기 위해 자체적인 서브 컴포지션을 생성합니다.  
벡터 컴포저블이 컴포지션될 때, 서브 컴포지션에 사용될 `VNode`라는 노드 타입을 발행합니다.  
이 `VNode`는 독립적인 경로(path) 또는 경로 그룹을 모델링하는 재귀 타입입니다.

```kotlin
@Composable
fun MenuButton(
    onMenuClick: () -> Unit
) {
    Icon(
        painter = rememberVectorPainter(image = Icons.Rounded.Menu),
        contentDescription = "Menu",
        modifier = Modifier.clickable(onClick = onMenuClick)
    )
}
```

여기서 흥미로운 점은, 일반적으로 벡터 그래픽을 표시할 때 `Image`, `Icon`과 같은 컴포저블 내에서 `VectorPainter`를 사용한다는 것입니다.
이는 벡터를 화면에 그리기 위해 사용되는데, 이 경우 외부의 컴포저블은 `Layout`이며, 따라서 해당 컴포지션에 `LayoutNode`를 내보낸다는 것을 의미합니다.

동시에, `VectorPainter`는 벡터를 위한 자체 서브 컴포지션을 생성하고, 이 서브 컴포지션은 상위 컴포지션과 연결되어 부모-자식 관계를 형성합니다.

<img alt="composition_and_subcomposition.png" src="composition_and_subcomposition.png" width="50%" title="Composition And Subcomposition"/>

이 설정을 통해 벡터 서브 트리(서브 컴포지션)는 `VNode`라는 노드 타입을 사용할 수 있습니다.

벡터는 서브 컴포지션을 통해 모델링되는데, 이는 벡터 컴포저블 호출(e.g : `rememberVectorPainter`) 내에서 부모 컴포지션의 `CompositionLocal`에 접근하는 것이 편리하기 때문입니다.
예를 들어, 테마 색상이나 밀도(density)와 같은 것들이 좋은 예시가 될 수 있습니다.

벡터를 위해 생성된 서브 컴포지션은 해당 `VectorPainter`가 부모 컴포지션에서 떠날 때마다 폐기됩니다.  
이는 `VectorPainter`를 포함하고 있는 컴포저블이 컴포지션에서 떠날 때마다 발생합니다.   
이후 장에서 컴포저블의 라이프사이클을 더 자세히 배우겠지만, 모든 컴포저블은 어느 시점에서 컴포지션에 들어가고, 다시 떠납니다.

이로써, 일반적인 Compose UI 애플리케이션(Android 또는 Desktop)에서 루트 컴포지션과 서브 컴포지션이 어떻게 구성되는지에 대해 더 자세히 이해했습니다.  

이제 플랫폼과의 통합 외 다른 측면, 즉 화면에 변경 사항을 반영하여 사용자에게 실제로 보여주는 것에 대해 이해할 차례입니다.

## Reflecting changes in the UI

앞서 초기 컴포지션과 재구성 프로세스를 통해 UI 노드가 어떻게 생성되고, 런타임에 전달되는지 알아봤습니다.  
이 시점에서 런타임은 Chapter3에서 배운 것처럼 자신의 역할을 수행합니다. 그러나, 이는 전체 프로세스의 일부에 불과합니다. 

생성된 모든 변경 사항을 실제 UI에 반영하여, 사용자가 이를 체험할 수 있도록 하는 통합 과정도 존재해야 합니다.  
이 프로세스는 흔히 노드 트리의 실체화(materialization)라고 불리며, 이는 클라이언트 라이브러리인 Compose UI의 책임입니다.

이전 장에서 이 개념을 간단히 소개했으니, 이번에는 더 자세히 살펴보겠습니다.

## Different types of Appliers

앞서, 런타임이 트리의 변경 사항을 최종적으로 실체화하는데 사용하는 추상화 계층으로 `Applier`를 소개했습니다.  
이 추상화는 런타임이 라이브러리가 사용되는 플랫폼에 대해 전혀 알 필요가 없도록 의존성을 역전시킵니다.

이러한 추상화 계층 덕분에 Compose UI와 같은 클라이언트 라이브러리는 자신의 `Applier` 구현을 연결할 수 있으며, 
이를 통해 플랫폼과의 통합에 사용할 노드 타입을 선택할 수 있습니다.

<img alt="img.png" src="compose_arch.png" width="80%"/>

> 상단의 `Applier`와 `AbstractApplier`는 Compose 런타임에 속합니다.  
> 하단의 상자들은 Compose UI에서 제공하는 여러 `Applier`의 구현들을 나열한 것입니다.

`AbstractApplier`는 Compose 런타임에서 제공하는 기본 구현으로, 다양한 `Applier` 간의 로직을 공유할 수 있도록 설계되었습니다. 
이 클래스는 방문한 노드를 스택에 저장하고, 현재 방문 중인 노드에 대한 참조를 유지하여, 어떤 노드에서 작업을 수행해야 하는지를 파악합니다.

트리에서 새로운 노드를 방문할 때마다, Composer는 `applier#down(node: N)`을 호출하여 `Applier`에게 이를 알립니다.  
그러면 해당 노드는 스택에 푸시되고, `Applier`는 해당 노드에서 필요한 작업을 수행할 수 있습니다.

방문자가 부모 노드로 돌아가야 할 때마다, Composer는 `applier#up()`을 호출하며, 그 결과 마지막으로 방문한 노드가 스택에서 제거(pop)됩니다.

아주 간단한 예시를 통해 이를 이해해보겠습니다. 다음과 같은 컴포저블 트리가 있다고 가정해봅시다:

```kotlin
Column {
    Row {
        Text("Some txt")
        if (condition) {
            Text("Some conditional txt")
        }
    }
    
    if (condition) {
        Text("Some more conditional txt")
    }
}
```

`condition`이 변경될 때마다, `Applier`는 다음과 같은 작업을 수행합니다:

- `Column`에 대한 `down` 호출을 받습니다.
- 그 다음, `Row`로 이동하기 위한 또 다른 `down` 호출을 받습니다.
- 그 다음, `condition`에 따른 선택적인 자식 `Text`의 삭제(또는 삽입) 작업을 수행합니다.
- 이후, 부모(`Column`)로 돌아가기 위한 `up` 호출이 이어집니다.
- 마지막으로, 두 번째 조건부 `Text`에 대한 삭제(또는 삽입) 작업을 수행합니다.

스택과 `down` 및 `up` 연산은 `AbstractApplier`에 포함되어 있어서, 자식 `Applier`들이 다루는 노드 타입에 상관없이 동일한 탐색 로직을 공유할 수 있습니다. 
이렇게 하면 노드 간의 부모-자식 관계가 자동으로 관리되므로, 특정 노드 타입은 트리를 탐색하기 위해 직접 부모-자식 관계를 유지할 필요가 없어집니다.
하지만, 필요에 따라 특정 노드 타입은 클라이언트 라이브러리의 특정 요구에 맞춰 자체적으로 부모-자식 관계를 구현할 수 있습니다.

> 위 예시는 실제로 `LayoutNode`의 사례이며, `LayoutNode`의 모든 작업이 컴포지션 중에 수행되는 것이 아니기 때문입니다.  
>
> 예를 들어, 노드를 다시 그려야 할 때, Compose UI는 부모 노드들을 따라 위로 탐색하면서, 해당 노드가 그리는 레이어를 생성한 노드를 찾아서 해당 레이어를 무효화합니다.
> 이 모든 작업은 컴포지션 외부에서 이루어지므로, Compose UI는 트리를 자유롭게 위아래로 탐색할 수 있는 수단이 필요합니다.

이 시점에서 Chapter3로 돌아가서, `Applier`가 노드 트리를 하향식(top-down), 상향식(down-top)으로 구축할 수 있는 방법을 설명했습니다.  
또한, 이러한 접근 방식이 성능에 미치는 영향과, 새 노드가 삽입될 때마다 알림을 받아야 하는 노드의 수에 따라 달라지는 성능 차이에 대해서도 설명했습니다.  
지금 약간 혼란스럽다면, Chapter3의 [Performance when building the node tree](Chapter%203%20%3A%20The%20Compose%20runtime.md#performance-when-building-the-node-tree) 섹션을 다시 읽어보는 것을 추천합니다.

이 내용을 다시 언급하는 이유는 Compose UI에서 노드 트리를 구축할 때 사용하는 두 가지 전략의 실제 예시가 존재하기 때문입니다.  
이들은 라이브러리에서 사용되는 두 가지 `Applier` 타입에 의해 구현됩니다.

Compose UI는 Android 플랫폼과 Compose 런타임을 통합하기 위해 두 가지 `AbstractApplier` 구현을 제공합니다:

- `UiApplier`
  - 대부분의 Android UI를 렌더링하는데 사용됩니다.
  - 노드 타입을 `LayoutNode`로 고정하여, 트리 내의 모든 레이아웃을 실체화합니다.
- `VectorApplier`
  - 벡터 그래픽을 렌더링하는데 사용됩니다.
  - 노드 타입을 `VNode`로 고정하여, 벡터 그래픽을 표현하고 실체화합니다.

> 현재 Android에서 제공되는 구현체는 이 두 가지뿐이지만, 특정 플랫폼에서 사용할 수 있는 구현체의 수는 곶어된 것이 아닙니다.  
> 앞으로 기존 노드 트리와는 다른 노드 트리를 표현해야 하는 필요성이 있다면, Compose UI에 더 많은 구현체가 추가될 수 있습니다.

방문하는 노드의 타입에 따라 다른 `Applier` 구현이 사용됩니다.  
예를 들어, 루트 컴포지션에 `LayoutNode`가 사용되고, 서브 컴포지션이 `VNode`가 사용된다면, 두 `Applier`가 모두 사용되어 전체 UI 트리를 실체화합니다.

두 `Applier`가 노드 트리를 구축하는데 사용하는 전략을 간단히 살펴보겠습니다.

`UiApplier`는 노드를 상향식으로 삽입하며, 이는 새로운 노드가 트리에 들어올 때 중복 알림을 피하기 위한 것입니다.  
이해를 돕기 위해 Chapter3에서 상향식 삽입 전략 다이어그램을 다시 가져오겠습니다:

<img alt="bottom_up.png" src="bottom_up.png" width="60%"/>

트리를 상향식으로 구축하는 방식은 먼저 A와 C를 B에 삽입한 다음, B 트리를 R에 삽입하여 트리를 완성합니다.  
이 방법은 새로운 노드가 삽입될 때마다 직접적인 부모에게만 알림을 보냅니다.  
상향식 전략은 일반적으로 중첩이 많이 발생하는 Android UI(특히, 오버드로(overdraw)가 문제가 되지 않는 Compose UI)에서, 알림을 받아야 하는 많은 상위 노드가 있을 때 특히 유용합니다.

반면에, `VectorApplier`는 트리를 하향식으로 구축하는 예시입니다.  
위에서 설명한 샘플 트리를 하향식으로 구축하려면, 먼저 B를 R에 삽입하고, 그 다음 A를 B에, 마지막으로 C를 B에 삽입합니다:

<img alt="up_bottom.png" src="up_bottom.png" width="60%"/>

하향식 전략에서는 새로운 노드를 삽입할 때마다, 해당 노드의 모든 상위 노드들에게 알림을 보내야 합니다.  
그러나 벡터 그래픽의 경우, 어떤 노드에게도 알림을 전파할 필요가 없기에, 두 전략 모두 성능 면에서 동등하게 유효합니다.  
따라서, 하향식과 상향식 전략 중 어느 것을 선택해도 큰 차이가 없습니다.  
새로운 자식 노드가 `VNode`에 삽입될 때는 해당 노드의 리스너에게 알림을 보내지만, 자식 노드나 부모 노드에게는 알림이 전파되지 않습니다.

이제 Compose UI 라이브러리가 사용하는 두 가지 `Applier` 구현에 대해 이해했으니, 이들이 최종적으로 UI에서 변경 사항(Changes)을 어떻게 실체화하는지 알아볼 차례입니다.