# Intrinsic measurements in Compose layouts

Compose 규칙 중 하나는 자식 요소를 한 번만 measuring 해야 하며, 만약 두 번 measuring 하게 되면, 런타임 예외가 발생합니다.  
하지만 자식 요소를 측정하기 전에, 자식 요소에 관한 정보가 필요한 상황이 있을 수 있습니다.

### 실제로 자식 요소가 측정되기 전에, Intrinsics 기능을 통해 자식 요소를 쿼리할 수 있습니다.

이는 컴포저블의 `intrinsicWidth` 또는 `intrinsicHeight`를 호출하여 가능합니다.

- `(min | max) IntrinsicWidth` : 이 너비가 주어졌을 떄, 콘텐츠를 그릴 수 있는 최소/최대 너비는 어느 정도인가?
- `(min | max) IntrinsicHeight` : 이 높이가 주어졌을 때, 콘텐츠를 그릴 수 있는 최소/최대 높이는 어느 정도인가?

예를 들어, `height`가 무한대인 `Text`의 `minIntrinsicHeight`를 호출하면 `Text`가 한 줄로 그려진 것처럼 텍스트의 `height`를 반환합니다.

> Intrinsics measurements를 호출하는 것은 자식을 두 번 측정하는 것이 아닙니다.  
> 자식들은 실제로 측정되기 전에, Intrinsics measurements를 통해 크기가 확인되며, 이 정보를 바탕으로 부모가 자식을 측정할 constraints를 계산합니다.

## Intrinsics in action

<img src="img.png" width="400">

위와 같이 화면에 Divider로 구분된 두 개의 텍스트를 표시하는 컴포저블을 만든다고 가정하겠습니다.
`Row`를 통해 두 개의 `Text`를 배치하고, `Divider`를 추가하여 두 텍스트를 구분합니다.
 
```kotlin
@Composable
fun ToTexts(
    modifier: Modifier = Modifier,
    text1: String,
    text2: String
) {
    Row(modifier = modifier) {
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(8.dp)
                .wrapContentWidth(Alignment.Start),
            text = text1
        )
        
        HorizontalDivider(
            modifier = Modifier
                .fillMaxHeight()
                .width(1.dp)
        )
        
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(8.dp)
                .wrapContentWidth(Alignment.End),
            text = text2
        )
    }
}
```

위 코드를 Preview로 확인한 결과, 다음과 같습니다.

<img src="img_1.png" width="400">

`Row`는 자식을 개별적으로 측정하며, `Text`의 높이를 통해 `Divider`를 제약할 수 없기 때문에 이러한 결과가 나타납니다.  

앞선 목표는 Divider가 주어진 높이에 맞게 공간을 차지하기를 원합니다.  
이를 위해 `height(IntrinsicSize.Min)`를 통해 `Divider`의 높이를 `Text`의 최소 높이로 설정할 수 있습니다.

```kotlin
@Composable
fun TwoTexts(modifier: Modifier = Modifier, text1: String, text2: String) {
    Row(modifier = modifier.height(IntrinsicSize.Min)) {
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(start = 4.dp)
                .wrapContentWidth(Alignment.Start),
            text = text1
        )
        HorizontalDivider(
            color = Color.Black,
            modifier = Modifier.fillMaxHeight().width(1.dp)
        )
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(end = 4.dp)
                .wrapContentWidth(Alignment.End),

            text = text2
        )
    }
}
```

이처럼, `Row` 컴포저블의 `minIntrinsicHeight`는 자식 컴포저블 중 가장 큰 `minIntrinsicHeight` 값이 됩니다.  
Divider 요소는 주어진 constraints가 없을 때 공간을 차지하지 않으므로, 기본 `minIntrinsicHeight` 값이 0입니다.  
`Text`의 `minIntrinsicHeight`는 주어진 너비에서의 텍스트 높이에 따라 결정됩니다.  
`Row`안에 여러 `Text` 요소가 있다면, `Row`는 이들 중 가장 큰 `minIntrinsicHeight` 값을 기준으로 높이 constraints를 계산합니다.  
그 후, Divider는 `Row`가 제공하는 `height` constraints에 맞춰 자신의 `height`를 확장하게 됩니다.

### Intrinsics in your custom layouts

Custom `Layout` 또는 `Layout` Modifier를 만들 때, intrinsics measurements는 자동으로 근사값을 기반으로 계산됩니다.  
따라서 모든 레이아웃에 대해 정확하지 않을 수 있습니다. 이러한 이유로, 해당 API는 기본값을 재정의할 수 있는 옵션을 제공합니다.

Custom `Layout`의 경우에는 `MeasurePolicy` 인터페이스를 생성할 떄, 
`minIntrinsicWidth`, `maxIntrinsicWidth`, `minIntrinsicHeight`, `maxIntrinsicHeight`를 재정의할 수 있습니다.

```kotlin
@Composable
fun MyCustomComposable(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier,
        measurePolicy = object : MeasurePolicy {
            override fun MeasureScope.measure(
                measurables: List<Measurable>,
                constraints: Constraints
            ): MeasureResult {
                // Measure and layout here
                // ...
            }

            override fun IntrinsicMeasureScope.minIntrinsicWidth(
                measurables: List<IntrinsicMeasurable>,
                height: Int
            ): Int {
                // Logic here
                // ...
            }
        }
    )
}
```

Custom `layout` modifier의 경우, `LayoutModifier` 인터페이스에서 관련 메서드를 재정의할 수 있습니다.

```kotlin
fun Modifier.myCustomModifier(
    /* ... */
) = this then object : LayoutModifier {

    override fun MeasureScope.measure(
        measurable: Measurable,
        constraints: Constraints
    ): MeasureResult {
        // Measure and layout here
        // ...
    }

    override fun IntrinsicMeasureScope.minIntrinsicWidth(
        measurable: IntrinsicMeasurable,
        height: Int
    ): Int {
        // Logic here
        // ...
    }
}
```