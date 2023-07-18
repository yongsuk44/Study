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