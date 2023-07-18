# Item 36: 상속 보다 구성을 우선시 할 것

상속은 객체 계층구조를 만드는 강력한 방법이지만 상속은 `is-a` 관계를 가진 객체 계층을 만들기 위해 설계 되었습니다.

`is-a` 관계가 명확하지 않으면 상속은 문제가 될 수 있습니다. 
코드의 단순한 추출이나 재사용이 필요한 경우 상속 대신 클래스 구성과 같은 가벼운 방법을 선택해야 합니다.

## 간단한 동작은 재사용

아래의 예시는 비슷한 동작을 가진 2개의 클래스가 있고 로딩 전,후에 프로그레스바를 표시하는 동일한 로직을 갖고 있습니다.

```kotlin
class ProfileLoader {
    fun load() {
        showProgressBar()
        loadProfile()
        hideProgressBar()
    }
}

class ImageLoader {
    fun load() {
        showProgressBar()
        loadImage()
        hideProgressBar()
    }
}
```

아래와 같이 공통 로직을 추출하기 위해 공통된 상위 클래스를 만드는 것이 일반적입니다.

```kotlin
abstract class LoaderWithProgress {
    fun load() {
        showProgressBar()
        innerLoad()
        hideProgressBar()
    }
    
    abstract fun innerLoad()
}

class ProfileLoader : LoaderWithProgress() {
    override fun innerLoad() {
        loadProfile()
    }
}

class ImageLoader : LoaderWithProgress() {
    override fun innerLoad() {
        loadImage()
    }
}
```

이 방법은 위와 같이 간단한 경우에 효과적이지만, 몇 가지 단점들이 존재합니다.

### 하나의 클래스만 상속할 수 있다.

상속을 사용하여 기능 추출 시, 많은 기능이 누적되어 거대해진 `BaseXXX` 클래스를 만들거나, 복잡하고 깊은 타입 계층을 만들게 될 수 있습니다.

### 클래스 상속 시 클래스의 모든 것을 가져와야 한다.

이는 필요로 하지 않은 기능과 메서드를 가진 클래스를 만들어 내게 됩니다. 이는 인터페이스 분리 원칙(Interface Segregation Principle)을 위반하는 행동 입니다.

### 상위 클래스의 기능을 사용하는 것은 매우 불명확 합니다.

이 메서드가 어떻게 동작하는지 이해하기 위해 여러 상위 클래스로 이동해서 확인하는 것은 좋지 않습니다.


이러한 이유들로 다른 방법을 고려해야 합니다.

그 중 하나로, 구성(Composition)을 사용할 수 있습니다. 구성은 객체를 프로퍼티로 보유하고 그 기능을 재사용하는 것을 의미 합니다.

아래는 상속 대신 구성을 사용하여 문제를 해결하는 예시 입니다.

```kotlin
class Progress {
    fun showProgress() { /* show progress */ }
    fun hideProgress() { /* hide progress */ }
}

class ProfileLoader {
    private val progress = Progress()
    
    fun load() {
        progress.showProgress()
        loadProfile()
        progress.hideProgress()
    }
}

class ImageLoader {
    private val progress = Progress()
    
    fun load() {
        progress.showProgress()
        loadImage()
        progress.hideProgress()
    }
}
```

구성은 모든 클래스에 구성 객체를 포함하고 그것을 사용해야 하기에 상속보다 더 복잡해 보일 수 있습니다.
이는 많은 개발자들이 상속을 선호하는 이유이기도 합니다. 

하지만 이 추가 코드는 무의미하지 않고 코드를 읽는 사람에게 프로그레스가 어떻게 사용되는지 알려주며, 작동 방식에 대해 더 많은 제어를 제공합니다. 

또한, 여러 기능을 추출하려는 경우 구성이 더 간편합니다. 
아래는 로딩이 완료되었다는 정보를 추출하는 예시 입니다.

```kotlin
class ImageLoader {
    private val progress = Progress() 
    private val finishedAlert = FinishedAlert()
    
    fun load() {
        progress.showProgress()
        loadImage()
        progress.hideProgress()
        finishedAlert.show()
    }
}
```

---

## Taking the whole package

상속 사용 시 상위 클래스로부터 모든 것(메서드, 예상되는 동작 및 그 외의 동작 등)을 가져오게 됩니다.
따라서 상속은 객체 계층 구조를 표현하는 데 좋은 방법이지만, 단순하게 일부 공통된 부분을 재사용하기 위해서는 사용될 필요가 없습니다.

위와 같이 단순한 일부 공통 부분을 재사용하기 위해서는 구성을 통해 필요로 하는 동작을 선택할 수 있습니다.

예를 들어 우리의 시스템에서 `Dog` 클래스를 표현하고, 이 `Dog`에 다음 2가지 동작을 추가하고 싶다고 가정해 봅시다.

```kotlin
abstract class Dog {
    open fun bark() { }
    open fun sniff() { }
}
```

그런 다음, `Dog`를 사용하여 짖을 수 있지만, 냄새를 맡을 수 없는 `RobotDog`를 만드면 다음과 같은 문제가 발생될 수 있습니다.

```kotlin
class Labrador: Dog()

class RobotDog: Dog() {
    override fun sniff() { 
        throw Error("Operation not supported")
    }
}
```

위 `RobotDog`는 필요하지 않은 메서드를 가지게 되므로 인터페이스 분리 원칙(interface segregation principle)을 위반합니다.
또한 상위 클래스의 동작을 깨뜨리게 되어 Liskov 치환 원칙(Liskov Substitution Principle)을 위반하는 행동입니다.

그리고 `RobotDog`가 계산을 할 수 있는 `Robot` 클래스로 표현되어야 할 경우 Kotlin에서는 다중 상속을 지원하지 않기에 문제가 될 수 있습니다.

```kotlin
abstract class Robot {
    open fun calculate() { }
}

class RobotDog : Dog(), Robot() // Error 
```

위와 같은 문제점 있는 설계와 제한 사항들은 구성을 사용하게 되면 해결할 수 있습니다.

- 재사용 하려는 기능을 선택할 수 있습니다.
- 타입 계층을 표현하는데에는 인터페이스를 사용하는 것이 더 안전합니다.
- 여러 인터페이스를 구현할 수 있습니다.

---

## Inheritance breaks encapsulation

클래스 확장 시 클래스가 어떻게 동작하는지 뿐만 아니라 내부 작동 방식에도 의존하기 때문에 상속은 캡슐화를 깨뜨릴 수 있습니다.

예를 들어 추가 요소의 수를 계산하는 `Set`이 필요하다고 가정하고 `HashSet`을 상속받아 구현하면 다음과 같습니다.

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

val counterList = CounterSet<String>()
counterList.addAll(listOf("A", "B", "C"))
print(counterList.elementsAdded) //6
```

하지만 위와 같이 상속을 통한 구현은 의도한 동작과 다르게 동작할 수 있습니다.

왜냐하면 `HashSet`의 `addAll`은 `add` 메서드를 내부적으로 사용하기 때문에 `addAll`을 통해 요소를 추가할 때 마다 카운터가 2번 증가하기 때문입니다.

이 문제를 해결하기 위해 `addAll`을 제거한다고 가정해봅시다.

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

하지만 이러한 해결책은 `HashSet`의 `addAll` 메서드가 `add` 메서드에 의존하지 않는 방식으로 변경될 경우 이 구현은 Java 업데이트와 함께 동작되지 않게 됩니다.

그러므로 상속 대신 구성을 사용하여 위 문제를 해결하는 것이 좋습니다.

```kotlin
class CounterSet<T> {
    private val innerSet = HashSet<T>()
    var elementsAdded = 0
        private set
    
    fun add(element: T): Boolean {
        elementsAdded++
        return innerSet.add(element)
    }
    
    fun addAll(elements: Collection<T>): Boolean {
        elementsAdded += elements.size
        return innerSet.addAll(elements)
    }
}

val counterList = CounterSet<String>()
counterList.addAll(listOf("A", "B", "C"))
print(counterList.elementsAdded) //3
```

위와 같은 구현은 `CounterSet`이 `Set`이 아니게 되어 다형성을 잃게 됩니다.

위 다형성 문제를 해결하기 위해 위임(delegation) 패턴을 사용할 수 있습니다.

위임 패턴은 특정 객체의 동작을 다른 객체에게 위임하게 만드는 것을 말하며, 특정 기능의 구현을 다른 객체에게 맡기는 방식으로 코드의 재사용성을 향상 시키고 더욱 깔끔하게 구조를 만들 수 있도록 합니다.

Kotlin에서는 위임 패턴을 사용하여 '특정 인터페이스를 구현하는 클래스'를 선언하면서, '동일한 인터페이스를 구현하는 다른 객체'에게 모든 메서드 호출을 자동으로 위임할 수 있습니다. 
이러한 과정을 인터페이스 위임(interface delegation)이라고 합니다.

```kotlin
class CounterSet<T>(
    private val innerSet: MutableSet<T> = mutableSetOf()
) : MutableSet<T> by innerSet {
    var elementsAdded = 0
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

위와 같은 상황처럼 다형성을 필요로 하고, 상속의 위험성이 존재할 때는 위임 패턴을 활용하는 것이 좋습니다.  
하지만 대부분의 경우 다형성은 필요로 하지않고 다른 방식으로 사용하게 됩니다. 
이럴 때는 위임 패턴 없이 구성을 사용하는 것이 더 적합합니다.

상속을 사용하면 상위 클래스의 내부 구현에 대한 지식이 필요하기에 클래스의 내부 구조가 외부로 노출 될 수 있습니다. 
이는 캡슐화 원칙을 위반하는 것으로 시스템의 보안성이 약화될 수 있습니다.

그러나 실제로 항상 이런 문제가 발생되는 것은 아니며, 대부분의 경우 클래스의 행동은 Contract(주석 및 단위 테스트)에 명시되어 있고, 하위 클래스는 이러한 계약에 의존하게 됩니다.
이런 방식으로 클래스의 행동을 예측할 수 있고, 문제가 생겨도 계약(contract)에 의해 보호받을 수 있습니다.

또한 클래스가 상속을 목적으로 설계된 경우에는 이러한 문제가 덜 발생됩니다. 
상속용으로 설계된 클래스는 내부 구현 대신 인터페이스에 초점을 맞추기에 하위 클래스가 상위 클래스의 내부 구현에 의존하게 되는 경우가 적습니다.

구성과 위임 패턴은 상속의 이런 문제점을 해결하는 데 도움이 됩니다.
구성은 클래스 간의 관계를 느슨하게 만들기에, 코드를 더 유연하고 재사용 가능하게 만들 수 있습니다.
또한, 위임 패턴 사용 시 구성을 사용하면서도 다형성을 보장할 수 있기에 유연성과 재사용성이 중요한 경우 구성과 대리 패턴을 사용하는 것이 좋습니다.
