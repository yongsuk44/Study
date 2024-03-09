상속은 강력한 기능이지만 'is-a' 관계의 객체 계층 구조를 만들도록 설계되었다.
'is-a'의 관계가 명확하지 않은 경우 상속은 문제가 될 있고 위험할 수 있다.
간단하게 코드를 재사용하거나 추출이 필요한 경우 상속은 신중하게 사용되어야 하며, 더 가벼운 대안인 'class composition'을 선호해야 한다.

## Simple behavior reuse

프로그래스바의 표시와 숨김이라는 비슷한 로직을 갖는 두 클래스가 있다고 가정해보자. 

```kotlin
class ProfileLoader {
    fun load() {
        // show progress
        // load profile
        // hide progress
    }
}

class ImageLoader {
    fun load() {
        // show progress
        // load image
        // hide progress
    }
}
```

위 코드를 대부분의 개발자는 아래와 같이 공통된 상위 클래스를 생성하여 공통 로직을 추출할 것이다.

```kotlin
abstract class LoaderWithProgress {
    fun load() {
        // show progress
        innerLoad()
        // hide progress
    }

    abstract fun innerLoad()
}

class ProfileLoader : LoaderWithProgress() {
    
    override fun innerLoad() {
        // load profile
    }
}

class ImageLoader : LoaderWithProgress() {
    
    override fun innerLoad() {
        // load image
    }
}
```

위와 같은 방식은 간단한 경우에는 문제 없이 동작하지만, 인지해야 할 몇 가지 중요한 단점들이 존재한다. 

클래스는 단 하나의 상위 클래스만 가질 수 있기에, 상속을 사용하여 함수를 추출하는 것은 많은 함수들을 축적하는 거대한 'BaseXXX' 클래스를 만들거나, 
다수의 상속 계층 구조를 갖는 경우와 같이 너무 깊고 복잡한 타입 계층구조를 만들게 될 수 있다. 

클래스를 확장하는 경우에는 상위 클래스의 모든 메서드와 필드를 상속받게 된다. 
이 때 만약, 상위 클래스에 하위 클래스가 필요로 하지 않는 메서드나 필드가 있다면, 이는 하위 클래스에 불필요한 기능이 추가되는 것이다.
이는 곧 인터페이스 분리 원칙(ISP)을 위반하는 행동이다.

상위 클래스의 기능을 사용하는 것이 명시적이지 않으면, 해당 기능이 어디에서 정의되었는지, 어떻게 동작하는지 파악하기 위해 상위 클래스의 코드를 참조해야 한다.
이러한 과정은 시간이 소요되고, 코드를 이해하는데 어려움을 겪게 된다.

---

이러한 단점으로 인해, 다른 방안을 고려했으며 그 중 하나는 'Composition'을 사용하는 것이다.  
'Composition'은 객체를 프로퍼티로 보유하고 그 기능을 재사용하는 것을 의미한다.

아래는 상속 대신 'Composition'을 사용하여 문제를 해결하는 예시이다.

```kotlin
class Progress {
    fun showProgress() { /* show progress */ }
    fun hideProgress() { /* hide progress */ }
}

class ProfileLoader {
    private val progress = Progress()

    fun load() {
        progress.showProgress()
        // load profile
        progress.hideProgress()
    }
}

class ImageLoader {
    private val progress = Progress()

    fun load() {
        progress.showProgress()
        // load image
        progress.hideProgress()
    }
}
```

'Composition'을 사용하려면 'composed object'를 모든 클래스에 포함시켜 사용해야 하기에 더 복잡해 보일 수 있어, 상속을 선호하는 경우가 많습니다.
그러나, 이런 추가적인 코드는 개발자에게 프로그레스의 동작 방식과 더 많은 제어권을 제공한다.
또한, 여러 기능을 추출하려는 경우 'Composition'을 사용하는 것이 더 간편하다.  

예를 들어, 아래는 정보를 불러오는 것이 완료됨을 알리는 알림 예시이다.

```kotlin
class ImageLoader {
    private val progress = Progress()
    private val finishedAlert = FinishedAlert()

    fun load() {
        progress.showProgress()
        // load image
        progress.hideProgress()
        finishedAlert.show()
    }
}
```

위 구현을 상속으로 대신하고자 한다면, 단 하나의 클래스만 확장할 수 있기에, 두 기능 모두를 하나의 상위 클래스에 배치해야 한다. 
이는 이러한 기능을 추가하기 위한 복잡한 타입 계층 구조를 만들게 되는데, 이러한 복잡한 계층 구조는 읽기 어렵고 수정하기 어렵다.

예를 들어, 일부 클래스에서는 알림이 필요하지만, 다른 하위 클래스에는 필요하지 않다면 어떻게 해야 할까?   
이러한 문제를 해결하기 위한 한 가지 방법은 '파라미터화된 생성자'를 사용하는 것이다.

```kotlin
abstract class InternetLoader(val showAlert: Boolean) {
    fun load() {
        // show progress
        innerLoad()
        // hide progress
        if (showAlert) {
            // show alert
        }
    }
    
    abstract fun innerLoad()
}

class ProfileLoader: InternetLoader(showAlert = true) {
    override fun innerLoad() {
        // load profile
    }
}

class ImageLoader: InternetLoader(showAlert = false) {
    override fun innerLoad() {
        // load image
    }
}
```

그러나, 위 방법은 하위 클래스에 불필요한 기능을 가져와 차단하기에 좋은 해결책이 아니다.  
상속을 사용하면 하위 클래스는 상위 클래스의 모든 메서드와 필드를 가져오게 되기에, 
만약 하위 클래스가 불필요한 기능을 차단할 수 없는 경우에는 이 문제가 더욱 복잡해진다.

---

## Taking the whole package

상속을 사용하면, 하위 클래스는 상위 클래스로부터 methods, expectations(contract), behavior 등을 상속 받는다.  
따라서 상속은 객체의 계층 구조를 나타내는데 훌륭한 방식이지만, 단지 몇몇 공통 부분을 재사용하기 위해서 사용될 필요는 없다.  
이처럼 몇몇 공통 부분을 재사용할 때에는 필요한 행동을 선택할 수 있는 'Composition'이 더 나은 방법이다. 

예를 들어, 시스템에서 'bark', 'sniff'를 할 수 있는 'Dog'를 표현하기로 했다고 가정해보자.

```kotlin
abstract class Dog {
    open fun bark() {}
    open fun sniff() {}
}
```

이 후에 'bark'는 가능하지만, 'sniff'를 할 수 없는 'RobotDog'를 만들어야 한다면 어떨까?

```kotlin
class Labrador : Dog()

class RobotDog : Dog() {
    
    override fun sniff() {
        throw Error("Operation not supported")  // Do you really want that?
    }
}
```

이러한 해결책은 'RobotDog'에는 불필요한 메서드가 있기에 인터페이스 분리 원칙(ISP)을 위반한다.  
또한, 상위 클래스의 행동을 깨뜨림으로써 리스코프 치환 원칙(LSP)도 위반한다.

다른 한편으로, 'RobotDog'은 'calculate' 메서드를 가지고 있어 'Robot' 클래스를 상속 받아야 한다면 어떻게 해야 할까?
Kotlin에서는 다중 상속을 지원하지 않기에 아래와 같이 문제가 될 수 있다.

```kotlin
abstract class Robot {
    open fun calculate() { /* ... */ }
}

class RobotDog : Dog(), Robot() // Error 
```

이러한 문제점과 제한 사항은 'Composition'을 사용하면 발생하지 않는다.  
'Composition'을 사용하면 개발자는 필요한 기능만을 선택하여 재사용할 수 있으며, 클래스가 불필요한 의존성을 갖지 않도록 한다.  
또한, 타입 계층 구조를 표현할 때 인터페이스를 사용하면, 한 클래스가 여러 인터페이스를 구현할 수 있기에 다중 상속의 이점을 얻을 수 있다.

상속은 상속 구조가 복잡해질 수록, 상위 클래스의 변경이 하위 클래스에 영향을 미칠 수 있으며, 이는 예측하기 어려운 동작을 유발할 수 있다. 

---

## Inheritance breaks encapsulation

상속을 사용하여 클래스를 확장하는 것은 해당 클래스의 공개 인터페이스뿐만 아니라 내부 구현에도 의존하게 만든다.  
즉, 상위 클래스의 내부 구현이 변경되면, 이러한 변경이 하위 클래스에 영향을 미칠 수 있으며, 이는 캡슐화 원칙을 위반하는 것이다.

아래는 위 내용에 대한 'Effective Java'에서의 한 예시이다.   
요소가 추가될 때마다 추가된 요소의 수를 계산하는 'CounterSet'이 필요하다고 가정하고 'HashSet'을 상속받아 구현하면 다음과 같을 것이다. 

```kotlin
class CounterSet<T> : HashSet<T>() {
    
    var elementsAdded = 0
        private set

    override fun add(element: T): Boolean {
        elementsAdded++
        return super.add(element)
    }

    override fun addAll(elements: Collection<T>): Boolean {
        elementsAdded += elements.size
        return super.addAll(elements)
    }
}
```

위 구현은 좋아보이지만, 올바르게 동작하지 않는다.

```kotlin
val counterList = CounterSet<String>()
counterList.addAll(listOf("A", "B", "C"))
print(counterList.elementsAdded) // 6
```

이유는 'HashSet'의 'addAll'은 내부에서 'add' 메서드를 사용하기 때문에, 'addAll'을 사용하면 추가된 각 요소에 대한 카운터가 2번 증가한다.
만약, 이 문제를 단순하게 해결하려면 사용자 정의 'addAll' 함수를 제거하면 된다.

```kotlin
class CountingSet<T> : HashSet<T>() {
    var elementsAdded = 0
        private set

    override fun add(element: T): Boolean {
        elementsAdded++
        return super.add(element)
    }
}
```

이런 해결책은 위험할 수 있다. 만약 Java를 만든 개발자들이 'HashSet.addAll'을 'add' 메서드에 의존하지 않도록 최적화하기로 결정한다면, 위와 같은 구현은 Java 업데이트와 함께 동작되지 않게 될 것이다.
더 나아가 위 구현을 기반으로 만들어진 라이브러리가 있다면, 마찬가지로 동작하지 않게 될 것이다.
Java 개발자들도 이런 상황을 인지하고 있기에, 위와 같은 변경은 매우 조심하긴 하지만, 
이는 라이브러리 제작자나 대규모 프로젝트를 진행하는 개발자들에게도 동일한 문제가 될 수 있다.

그러므로, 이러한 문제를 해결할 때는 상속 대신 'Composition'을 사용하는 것이 좋다.

```kotlin
class CounterSet<T> {
    
    private val innerSet = HashSet<T>()
    var elementsAdded = 0
        private set

    fun add(element: T) {
        elementsAdded++
        innerSet.add(element)
    }

    fun addAll(elements: Collection<T>) {
        elementsAdded += elements.size
        innerSet.addAll(elements)
    }
}

val counterList = CounterSet<String>()
counterList.addAll(listOf("A", "B", "C"))
print(counterList.elementsAdded) //3
```

이런 방식을 채택하게 되면 한 가지 문제가 발생하게 되는데, 바로 'CounterSet'이 'Set'의 성질을 잃게 된다는 점이다.  
즉, 'CounterSet'이 'Set' 인터페이스를 따르는 객체로서의 역할을 하지 못하게 된다는 것이다. (다형성을 잃게 된다.)

이를 해결하기 위해서는 'delegation pattern'을 사용할 수 있다.  
'delegation pattern'은 클래스가 어떤 인터페이스를 구현하고, 동일한 인터페이스를 구현하는 객체를 구성한 뒤,
인터페이스에 정의된 메서드를 이 구성된 객체로 전달하는 방식이다. 이렇게 전달되는 메서드들을 'forwarding methods'라고 한다.

```kotlin
class CounterSet<T>: MutableSet<T> {
    
    private val innerSet = HashSet<T>()         // Composition
    var elementsAdded = 0
        private set

    override fun add(element: T): Boolean {
        elementsAdded++
        return innerSet.add(element)            // Forwarding method
    }

    override fun addAll(elements: Collection<T>): Boolean {
        elementsAdded += elements.size
        return innerSet.addAll(elements)        // Forwarding method
    }
    
    override val size: Int 
        get() = innerSet.size
    
    override fun contains(element: T): Boolean =
        innerSet.contains(element)
    
    override fun containsAll(elements: Collection<T>): Boolean =
        innerSet.containsAll(elements)
    
    override fun isEmpty(): Boolean =
        innerSet.isEmpty()
    
    override fun iterator(): MutableIterator<T> = 
        innerSet.iterator()
    
    override fun clear() = 
        innerSet.clear()
    
    override fun remove(element: T): Boolean = 
        innerSet.remove(element)
    
    override fun removeAll(elements: Collection<T>): Boolean =
        innerSet.removeAll(elements)
    
    override fun retainAll(elements: Collection<T>): Boolean =
        innerSet.retainAll(elements)
    
}
```

이제 또 다른 문제는 많은 수의 'forwarding methods'를 구현해야 한다는 것이다. (위 경우에는 9개의 메서드)  
다행히도, Kotlin은 이런 상황을 돕기 위해 'Interface delegation'을 지원한다.

특정 객체에 인터페이스의 역할을 위임하면, Kotlin은 자동으로 컴파일 과정에서 필요한 'forwarding methods'를 생성해준다.

```kotlin
class CounterSet<T>(
    private val innerSet: MutableSet<T> = mutableSetOf()
) : MutableSet<T> by innerSet {                 // Interface delegation
    
    var elementsAdded = 0                       // Composition
        private set

    override fun add(element: T): Boolean {
        elementsAdded++
        return innerSet.add(element)            // Forwarding method
    }

    override fun addAll(elements: Collection<T>): Boolean {
        elementsAdded += elements.size
        return innerSet.addAll(elements)        // Forwarding method
    }
}
```

위와 같이 다형성을 필요로 하고, 상속을 사용하는 것이 위험할 때에는 'delegation pattern'을 사용하는 것이 좋다.  
그러나, 대부분의 경우 다형성이 필요하지 않거나 다른 방식으로 사용된다.  
이런 경우에는 'delegation pattern' 보다는 'Composition'을 사용하는 것이 코드를 더 쉽게 이해할 수 있고 더 유연하다.

상속이 캡슐화를 깨뜨리는 점은 보안상의 문제를 일으킬 수 있다. 
하지만, 대부분의 경우 클래스의 행동은 'contract'(주석, 단위 테스트 등)에 명시되어 있거나,
상속을 염두에 두고 메서드를 설계한 경우 하위 클래스에서 특정 구현에 의존하지 않도록 할 수 있다.  
그럼에도 불구하고, 상속 대신 'Composition'을 사용하는 것은 더 많은 유연성과 재사용성을 제공하기에 때문이다.

---

## Restricting overriding

상속을 위해 설계되지 않은 클래스를 확장하지 못하도록 하려면 'final'로 선언하면 된다. 
특별한 이유로 상속을 허용해야 하는 경우에도 기본적으로 모든 메서드들은 'final'이기에, 'override'를 허용하고 싶은 메서드만 'open'으로 설정하면 된다.


```kotlin
open class Parent {
    fun a() {}
    open fun b() {}
}

class Child: Parent() {
    override fun a() {} // Error : final a cannot be overridden
    override fun b() {}
}
```

이런 메커니즘을 적절하게 활용하면 상속을 위해 설계된 메서드만 'open'으로 설정할 수 있다.  
또한, 메서드를 'override' 하면 모든 하위 클래스에서 해당 메서드를 'final'로 설정할 수 있다는 점을 기억해야 한다.

```kotlin
open class ProfileLoader: InternetLoader() {
    final override fun loadFromInternet() {
        // load profile
    }
}
```

이렇게 하면, 하위 클래스에서 'override' 할 수 있는 메서드의 수를 제한할 수 있다.