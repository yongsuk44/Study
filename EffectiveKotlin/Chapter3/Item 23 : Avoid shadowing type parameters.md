'shadowing'으로 인해 같은 이름의 프로퍼티와 파라미터를 정의 할 수 있다.  
이때, 로컬 파라미터가 외부 범위 프로퍼티를 가릴 수 있기에 혼동되지 않으며, 이런 상황은 개발자들에게 익숙하고 눈에 잘 띄기에 IDE가 경고를 하지 않는다.

```kotlin
class Forest(val name: String) {
    fun addTree(name: String) { /* ... */  }
}
```

반면에, 클래스 타입 파라미터를 함수 타입 파라미터로 섀도잉하는 경우에도 IDE가 경고를 하지 않는다.  
하지만, 이런 상황은 눈에 잘 띄지 않아 심각한 문제로 이어질 수 있다. 

보통 이런 실수는 제네릭의 동작 방식을 잘 이해하지 못하는 개발자로 인해 발생할 수 있다.

```kotlin
interface Tree
class Birch: Tree
class Spruce: Tree

class Forest<T: Tree> {
    fun <T: Tree> addTree(tree: T) { /* ... */ }
}
```

위 코드의 문제점은 'Forest'와 'addTree'의 타입 파라미터가 서로 독립적으로 되어있다는 점에 있다.  
즉, 'Forest'의 인스턴스가 특정 타입의 'Tree'만을 다루도록 제네릭을 사용하여 'type-safe'를 보장하려 했지만,  
'addTree'에서 다른 타입의 'Tree'도 추가할 수 있게 함으로써 이런 의도를 무시하고 있다.

```kotlin
val forest = Forest<Birch>()
forest.addTree(Birch())
forest.addTree(Spruce())
```

이런 상황은 바람직하지 않으며, 개발자들이 이해하기 어려울 수 있다.  
이에 대한, 한 가지 해결책은 'addTree'가 클래스 타입 파라미터 T를 사용하는 것이다.

이로 인해, 'Forest' 인스턴스에 추가되는 'Tree'의 타입이 'Forest' 인스턴스 생성 시 지정된 타입과 일치 해야하는 것을 보장할 수 있게 된다. 

```kotlin
class Forest<T: Tree> {
    fun addTree(tree: T) { /* ... */ }
}

val forest = Forest<Birch>()
forest.addTree(Birch())
forest.addTree(Spruce()) // Error: Type mismatch
```

만약, 새로운 타입 파라미터를 추가해야 하는 경우에는 이름을 다르게 지정하는 것이 좋다.  
유의할 점은, 새로운 타입 파라미터는 기존 타입 파라미터의 하위 타입으로 제한될 수 있다.

```kotlin
class Forest<T: Tree> {
    fun <ST: T> addTree(tree: ST) { /* ... */ }
}
```