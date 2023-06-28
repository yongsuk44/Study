# Item 23: 타입 파라미터 섀도잉을 하지말자

섀도잉으로 인해 속성(property)와 파라미터(parameter)를 같은 이름으로 정의할 수 있습니다. 
로컬 파라미터가 외부 범위의 속성을 섀도잉합니다. 

개발자들에게는 아래 상황이 비교적 눈에 잘 띄기 때문에 특별한 경고를 하지 않습니다.

```kotlin
class Forest(val name: String) {
    fun addTree(name: String) { /* ... */  }
}
```

그러나, 함수의 타입 파라미터가 클래스의 타입 파라미터를 가리는 상황도 발생할 수 있습니다. 
이 경우는 상대적으로 덜 눈에 띄며, 심각한 문제를 발생시킬 수 있습니다.  
이런 실수는 주로 제네릭의 작동 방식을 제대로 이해하지 못한 개발자들이 범하는 경우가 많습니다.

```kotlin
interface Tree
class Birch: Tree
class Spruce: Tree

class Forest<T: Tree> {
    fun <T: Tree> addTree(tree: T) { /* ... */ }
}
```

`Forest`와 `addTree`의 타입 파라미터는 서로 독립적인 부분이 문제가 될 수 있습니다.

```kotlin
val forest = Forest<Birch>()
forest.addTree(Birch())
forest.addTree(Spruce())
```

위와 같은 상황은 좋지 않으며 혼란스러울 수 있습니다.
해결책으로는 `addTree`가 클래스 타입 파라미터 T를 사용하도록 하는 것입니다.

```kotlin
class Forest<T: Tree> {
    fun addTree(tree: T) { /* ... */ }
}

val forest = Forest<Birch>()
forest.addTree(Birch())
forest.addTree(Spruce()) // Error: Type mismatch
```

새로운 타입 파라미터를 도입해야 하는 경우, 다른 이름을 사용하는 것이 더 좋습니다. 또한, 다른 타입 파라미터의 하위타입으로 제한될 수 있습니다.

```kotlin
class Forest<T: Tree> {
    fun <U: T> addTree(tree: U) { /* ... */ }
}
```