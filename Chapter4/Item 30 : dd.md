# Item 30: Minimize elements visibility

## API 설계 시 간결하게 만드는 것을 선호하는 이유

### 인터페이스가 작을수록 학습하고 유지 관리하기 쉽습니다.
수십 가지의 기능이 있는 클래스보다, 몇 가지 기능만 있는 클래스를 이해하는 것이 더 간단하며 유지보수도 더 간단합니다.
변경사항 적용 시 대체로 전체 클래스를 이해해야 하는데, 보이는 요소가 적을수록 유지하고 테스트할 부분이 적어집니다. 

### API 변경 시 기존 요소를 숨기는것보다 새로운 것을 공개하는 것이 더 간단합니다.
공개된 모든 요소들은 API의 일부이며 외부에서 사용될 수 있습니다.
요소가 공개된 기간이 길수록 외부에서 사용될 가능성이 높으며, 이러한 요소들을 변경하는 것은 사용 사례를 업데이트 해야하므로 복잡합니다.

가시성을 제한하는 것은 더욱 어렵습니다. 만약 가시성을 제한하는 경우, 사용 사례를 신중히 고려하고 대체 방안을 제시해줘야 합니다.
또한 다른 개발자가 구현한 경우, 대체 방안을 제시하는 것도 어려울 수 있습니다.

만약 공개 라이브러리인 경우, 가시성을 제한하는 것은 기존 사용하던 개발자들이 불편할 수 있습니다.
그들은 자신들이 구현한 방식을 수정해야하고, 구현한 코드가 개발된지 얼마나 된지 알 수 없을 수도 있기에 수정을 하는 것에도 어려움이 생길 수 있습니다.
따라서 처음부터 개발자들에게 더 작은 API를 사용하도록 하는 것이 훨씬 좋은 방법 입니다.

### 클래스 상태를 나타내는 속성들이 외부에서 변경될 경우, 클래스는 자신의 상태를 책임질 수 없습니다.
클래스 상태에 대한 특별한 가정이 있다면, 이 클래스는 이를 만족해야 합니다.
만약 특별한 가정을 만족해야 하는 상태가 외부에서 변경된다면, 내부 규약을 모르는 사람이 외부에서 변경할 수 있기 때문에 클래스는 불변성을 보장할 수 없습니다.

아래 코드를 보면 `elementsAdded`의 `setter`의 가시성을 적절히 제한하였습니다.   
만약 `setter`에 `private`이 없다면 누군가가 외부에서 이 값을 임의로 변경할 수 있고 
우리는 이 값이 실제로 얼마나 많은 요소가 추가되었는지를 정확히 표현할 수 없어 신뢰할 수 없을 것입니다.

```kotlin
class CounterSet<T>(
    private val innerSet: MutableSet<T> = setOf()
) : MutableSet<T> by innerSet {
    var elementsAdded: Int = 0
        private set

    override fun add(element: T): Boolean {
        elementsAdded++
        return innerSet.add(element)
    }

    override fun addAll(elements: Collection<T>): Boolean {
        elementsAdded += elements.size
        return innerSet.addAll(elements)
    }
}
```

For many cases, it is very helpful that all properties are encapsulated by default in Kotlin because we can always restrict the visibility of concrete accessors.

Protecting internal object state is especially important when we have properties depending on each other. For instance, in the below mutableLazy delegate implementation, we expect that if initialized is true, the value is initialized and it contains a value of type T. Whatever we do, setter of the initialized should not be exposed, because otherwise it cannot be trusted and that can lead to an ugly exception on a different property.

Kotlin에서는 모든 속성이 기본적으로 캡슐화되어 있는것이 구체적으로 속성에 대한 접근자의 가시성을 제한할 수 있기 때문에 매우 유용합니다.

속성들이 서로에게 의존하는 경우 내부 객체 상태를 보호하는 것은 굉장히 중요합니다.  

아래 코드에서 `initialized`가 `true`일 경우 `value`는 초기화 되어 `T`타입의 값을 포함하고 있다고 가정하고 있습니다.
만약 `initialized`가 노출되어 있어 `setter`를 통해 변경이 가능하다면, `value`의 상태를 신뢰할 수 없게 되고 이는 다른 속성에 대한 예외를 발생시킬 수 있습니다.

```kotlin
class MutableLazyHolder<T>(val initializer: () -> T) {
    private var initialized = false
    private var value: Any = Any()

    override fun get(): T {
        if (!initialized) {
            value = initializer()
            initialized = true
        }
        
        return value as T
    }
    
    override fun set(value: T) {
        this.value = value
        initialized = true
    }
}
```

### 제한적인 가시성을 가질 때 클래스의 변경을 추적하는 것이 더 간단합니다.
이는 속성 상태를 이해하는 것을 더 쉽게 만들며, 동시성을 다루는 경우에 중요합니다.
상태 변화는 병렬 프로그래밍에 문제를 일으키고, 가능한 많이 제어하고 제한하는 것이 좋습니다.
