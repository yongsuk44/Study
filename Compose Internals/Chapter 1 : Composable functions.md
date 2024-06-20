## The nature of Composable functions

Composable 함수는 `@Composable` 어노테이션이 붙은 Kotlin 함수입니다.   
이 어노테이션을 붙이면 함수가 데이터를 composable 트리의 노드로 변환합니다.   
쉽게 말해, UI를 구성하는 작은 조각들이라고 생각하면 됩니다. 

```kotlin
@Composable
fun NamePlate(name: String) { ... }
```

여기서 `NamePlate` 함수는 `name`을 받아 UI 요소로 변환합니다.   
`@Composable`은 이 함수가 데이터를 UI 요소로 변환할 것임을 나타냅니다.

Composable 함수는 실행될 때 'emitting'이라는 작업을 통해 composable 트리에 노드를 추가합니다.  
이는 Composition 단계에서 이루어지며, 트리의 구조를 최신 상태로 유지합니다.  
쉽게 말해, UI를 구성하는 새로운 요소가 생길 때마다 트리에 추가된다고 생각하면 됩니다.

대부분의 Composable 함수는 `Unit`을 반환하지만, 일부는 값을 반환합니다.   
예를 들어, `remember`는 연산의 결과를 'memoize'(기억)하고 반환합니다:

```kotlin
@Composable
fun NamePlate() {
    val name = remember { generateName() }
    Text(name)
}
```

여기서 `remember`는 `generateName()`의 결과를 기억하여 재사용합니다.   
이는 함수가 실행될 때마다 동일한 결과를 반환하도록 보장합니다.

`@Composable`은 함수에 몇 가지 제약과 속성을 부여하여, Compose 런타임이 다양한 최적화를 수행할 수 있게 합니다.   
예를 들어, 병렬로 함수를 실행하거나, 변경된 부분만 스마트하게 재구성(Smart Recomposition)하는 등의 최적화가 가능합니다. 이는 UI 성능을 향상시키는 데 매우 유용합니다.

일반적으로, 런타임 최적화는 코드의 특정 조건과 동작에 대한 확신을 통해 가능해집니다.   
예를 들어, 코드가 서로 독립적인지, 병렬로 실행 가능한지 등의 조건을 확신하면, 더 효율적인 실행 전략을 사용할 수 있습니다. 
이는 전체 프로그램의 성능을 향상시키는 데 도움이 됩니다.

