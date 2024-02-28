API를 설계할 때 가능한 간결하게 만드는 데에는 여러 가지 이유가 있으며 다음과 같다. 

**인터페이스가 작을수록 학습이 쉽고, 유지 보수하기가 쉽다.**  
예를 들어, 수십 가지의 메서드를 가진 클래스 보다 몇 가지 메서드만을 가진 클래스가 이해하기 더 쉽다.  
또한, 클래스 전체를 이해한 상태에서 변경사항을 적용해야 하는 경우가 많은데, 이 때 표시되는 요소가 적을수록 유지보수 및 테스트할 항목이 줄어들기에 유지보수가 더 쉬워진다. 

**변경사항을 적용하고 싶을 때, 기존 요소를 숨기는 것 보다는 새로운 사항을 노출하는 것이 훨씬 더 쉽다.**  
한 번 공개된 API 요소는 외부에서 사용될 수 있으며, 외부에 오래 노출될수록 더 많이 사용될 수 있다. 
이에 따라 이러한 요소를 변경하거나 제거하려고 할 때 외부 코드에 대한 광범위한 업데이트가 필요할 수 있다.
여기에 더해, 기존에 공개된 API 요소의 가시성을 제한하는 것은 이미 사용 중인 기능에 대한 대체 방안을 제공 해야함으로 더욱 어렵다. 
또한, 비지니스 요구 사항이 어떤 것이었는지 시간이 지난 후에는 파악하기가 어렵다.  
공개 라이브러리에서 일부 요소의 가시성을 제한하면, 이를 사용하는 사용자들이 불만을 가질 수 있다. 
왜냐하면 사용자들의 코드를 조정해야 하고, 코드가 개발된지 몇 년이 지난 후에 대체 솔루션을 구현해야 하는 문제에 직면하게 될 것이다.
이러한 이유로, 처음부터 사용자들이 더 작은 API를 사용하도록 강제하는 것이 훨씬 낫다.

**클래스의 상태를 나타내는 프로퍼티가 외부에서 변경될 수 있으면, 클래스는 자신의 상태를 책임질 수 없다.**  
클래스는 자신이 만족하는 상태에 대한 전제가 있을 수 있다. 만약 이 상태를 외부에서 직접 변경할 수 있다면, 현재 클래스는 '내부 계약'을 모르는 누군가에 의해 외부적으로 변경될 수 있기에 그 불변성을 보장할 수 없다. 
아래 'CounterSet' 클래스를 보면, 'elementsAdded' 프로퍼티의 'setter' 가시성을 적절히 제한하고 있다. 
이러한 가시성 제한이 없으면, 누군가가 외부에서 어떤 값으로든 변경할 수 있고, 이로 인해 실제로 얼마나 많은 요소가 추가되었는지를 신뢰할 수 없을 것이다.

```kotlin
class CounterSet<T>(
    private val innerSet: MutableSet<T> = setOf()
): MutableSet<T> by innerSet {
    
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

Kotlin에서 모든 프로퍼티가 기본적으로 캡슐화되어 있다는 점은 항상 구체적인 접근자의 가시성을 제한할 수 있기에 매우 유용하다.

내부 객체 상태를 보호하는 것은 프로퍼티가 서로 의존성을 가질 때 특히 중요하다.  
예를 들어, 아래의 'mutableLazy'의 구현을 위임할 때, 'initialized'가 'true'이면 'value'가 초기화되고, 'T' 타입의 값을 포함하고 있다고 예상한다.
무엇을 하든 'initialized'의 'setter'가 노출되어서는 안된다. 그렇지 않으면 신뢰할 수 없게 되고, 다른 프로퍼티에서 예외를 일으킬 수 있다.

```kotlin
class MutableLazyHolder<T>(val initializer: () -> T) {
    
    private var value: Any = Any()
    private var initialized = false

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

**클래스가 더 제한된 가시성을 가지면 클래스가 어떻게 변경되는지 더 쉽게 추적할 수 있다.**  
이는 프로퍼티 상태를 이해하기 쉽게 만들며, 동시성을 다룰 때 매우 중요하다.
상태의 변경은 병렬 프로그래밍에 있어 문제가 되기에 가능한 제어하고 제한하는 것이 좋다.

## Using visibility modifiers

내부의 희생 없이 외부에서 보기에 더 작은 인터페이스를 구현하기 위해 요소의 가시성을 제한한다.
일반적으로 요소가 표시되어야 할 이유가 없는 경우 숨기는 것이 바람직하다.
따라서 덜 제한적인 가시성 타입을 사용해야 할 이유가 없다면 클래스와 요소의 가시성을 최대한 제한적으로 설정하는 것이 좋다.
이를 위해 'visibility modifier'를 사용한다.

클래스 멤버의 경우, 동작과 함께 사용할 수 있는 4가지 'visibility modifier'는 다음과 같다.

- public (default) - 선언된 클래스를 볼 수 있는 클라이언트라면 어디에서든 보인다.
- private - 클래스 내부에서만 보인다.
- protected - 클래스 내부와 하위 클래스에서만 보인다.
- internal - 해당 모듈 내부에서만 보이며, 선언된 클래스를 볼 수 있는 클라이언트에게 보인다.

'Top-level 요소(함수, 프로퍼티, 클래스 등)에는 다음 3가지 'visibility modifier'가 있다.

- public (default) - 모든 곳에서 보인다.
- private - 같은 파일 내에서만 보인다.
- internal - 같은 모듈 내에서만 보인다.

모듈은 패키지와 다르며, Kotlin에서는 함께 컴파일된 Kotlin 소스 집합으로 정의된다.

- Gradle 소스 집합
- Maven 프로젝트
- IntelliJ IDEA 모듈
- 'Ant' 작업의 한 번의 호출로 컴파일된 파일 집합

만약, 모듈이 다른 모듈에 의해 사용될 수 있다면, 외부에 노출시키고 싶지 않은 'public' 요소들의 가시성을 'internal'로 변경해야 한다.
요소가 상속을 위해 설계 되었고, 클래스와 하위 클래스에서만 사용된다면, 'protected'로 만들어야 한다.
같은 파일이나 클래스 내에서만 요소를 사용한다면, 'private'로 만들어야 한다.
이 관례는 Kotlin에서도 지원되며, 요소가 오직 로컬에서만 사용될 경우 가시성을 'private'으로 제한하도록 제안한다.

![img.png](visibility_private.png)

이 규칙은 데이터를 보관하기 위해 설계된 클래스(DTO)의 프로퍼티에는 적용되어선 안된다.
만약, 서버에서 나이를 포함한 사용자 정보를 반환하고, 이를 파싱하기로 결정했다면, 현재 사용하지 않는다고 해서 이를 숨길 필요가 없다.
이는 사용되기 위해 존재하는 것이므로 표시하는 것이 더 낫다. 만약, 필요하지 않다면, 이 프로퍼티를 완전히 제거하는게 좋다.

```kotlin
class User(
    val name: String,
    val age: Int
)
```

API를 상속할 때 큰 제약 중 하나는, 멤버를 오버라이딩함으로써 그 멤버의 가시성을 제한할 수 없다는 점이다.  
이는 하위 클래스가 항상 그 상위 클래스로 사용될 수 있기 때문이다. 이는 상속 대신 구성(Composition)을 선호하는 또 다른 이유 중 하나이다.