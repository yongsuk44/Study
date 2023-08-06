# Item 27: 추상화 사용으로 인한 코드 보호

함수나 클래스를 추상화하여 실제 코드를 숨기는것은 사용자로부터 세부 사항을 보호하는것만이 아닌, 나중에 자유롭게 코드를 변경할 수 있도록 합니다.
예를 들어, 정렬 알고리즘을 함수로 만들고, 이를 사용하는 곳을 변경하지 않고도 정렬 알고리즘을 최적화하여 성능 향상을 할 수 있습니다.

사용자가 변경사항에 대해 모르게 자동차 제조사와 정비사들은 자동차 형태를 유지하면서 자동차의 모든 부분을 변경할 수 있습니다.
이는 제조사가 자유롭게 더 안전하거나, 더 친환경적인 자동차를 만들기 위해 더 많은 센서나 기능을 추가하는 등의 작업을 할 수 있음을 의미합니다.

## 상수를 이용한 추상화

상수를 사용하는 것은 코드를 더욱 이해하기 쉽게 만들며, 나중에 값이 변경되어야 할 때 관리를 용이하게 합니다.

패스워드 유효성 검사의 예시로 처음 코드 입니다.

```kotlin
fun isPasswordValid(text: String): Boolean {
    if (text.length < 7) return false
}
```

여기서 '7'이라는 값은 문맥에서 이해할 수 있지만, 이를 상수로 추출하면 더 명확해집니다.

```kotlin
const val MIN_PASSWORD_LENGTH = 7

fun isPasswordValid(text: String): Boolean {
    if (text.length < MIN_PASSWORD_LENGTH) return false
}
```

이렇게 하면 최소 패스워드 길이를 수정하는 것이 더 쉬워집니다. 검증 로직을 이해할 필요 없이, 상수값만 바꾸면 됩니다.
특히 한 번 이상 사용되는 값은 추출하는 것이 중요합니다.

예를 들어, 동시에 데이터베이스에 연결할 수 있는 스레드의 최대 수는 다음과 같이 설정할 수 있습니다.

```kotlin
val MAX_THREADS = 10
```

이렇게 추출하면 필요할 때 쉽게 변경할 수 있습니다. 만약 이 숫자가 프로젝트 전체에 퍼져 있다면 변경하는 것이 어려울 것입니다.

> 상수를 추출하는 것은 상수에 이름을 부여하고, 나중에 값을 변경하기 쉽게 만듭니다.

---

## 함수를 이용한 추상화

어플리케이션 개발 중 사용자에게 토스트 메시지를 보여줄 때 코드로 구현하면 아래와 같습니다.

```kotlin
Toast.makeText(context, "Hello World!", Toast.LENGTH_SHORT).show()
```

만약 위 토스트 메시지를 반복적으로 사용하게 되면 위 코드를 아래와 같이 간단한 함수로 추출할 수 있습니다.

```kotlin
fun Context.toast(
    msg: String,
    duration: Int = Toast.LENGTH_SHORT
) {
    Toast.makeText(this, msg, duration).show()
}
```

위와 같이 공통된 알고리즘을 함수로 추출하면 매번 토스트를 표시하는 방법을 기억할 필요가 없습니다.  
만약 토스트를 표시하는 방식이 변경될 경우에도 이 방법이 유용할 것입니다.  

그럼에도 불구하고, 예측하지 못한 변경사항에는 대응하기 어렵습니다.

예를 들어, 토스트 메시지 대신 스낵바라는 다른 방식으로 메시지를 표시하고자 한다면 함수 내부의 구현을 변경하고 이름을 바꿀수 밖에 없습니다.

```kotlin
fun Context.snackbar(
    msg: String,
    length: Int = Toast.LENGTH_SHORT
) {
    // ...
}
```

하지만, 이런 방식에는 다음과 같은 문제점이 있습니다.
1. 함수의 이름을 변경하는 것은 매우 위험할 수 있다. 특히, 함수에 의존하는 다른 모듈이 있다면 더욱 더 위험합니다.
2. 파라미터는 쉽게 자동으로 변경되지 않기에 Toast API를 계속 사용해야 하는 문제가 있습니다.

이처럼 스낵바를 표시할 때 토스트의 필드를 의존해서는 안됩니다.  
또한, 모든 사용처를 스낵바의 `enum`으로 변경하는 것 또한 문제가 될 수 있습니다.

```kotlin
fun Context.snackbar(
    msg: String,
    duration: Int = Snackbar.LENGTH_LONG
) {
    // ...
}
```

메시지를 어떻게 표시할 것인지에 대한 방법이 변경될 수 있다는 것을 이해하고 있으면 
중요한 점은 메시지가 어떻게 표시되는 것이 아닌, 사용자에게 메시지를 표시하려는 기능 자체에 포커스를 두어야 합니다.

개발자들에게 메시지를 표시하는 더 추상적인 방법이 필요하므로 토스트를 표시하는 것을 `showMessage` 함수를 만들어 한번 더 추상화 시킬 수 있습니다.

```kotlin
fun Context.showMessage(
    message: String,
    duration: MessageLength = MessageLength.LONG
) {
    val toastDuration = when (duration) {
        SHORT -> Toast.LENGTH_SHORT
        LONG -> Toast.LENGTH_LONG
    }

    Toast.makeText(this, message, toastDuration).show()
}

enum class MessageLength { SHORT, LONG }
```

이렇게 가장 많이 변경된 부분은 함수의 이름입니다. 일부 개발자들은 이름이 그저 레이블에 불과하다며 이러한 변화의 중요성을 간과할 수 있습니다.   
하지만 컴파일러의 관점에서는 그럴 수 있지만, 개발자의 관점에서는 아닙니다.

함수는 추상화를 나타내며, 함수의 시그니처는 이 추상화가 무엇인지를 알려주기에 의미있는 이름은 매우 중요합니다.

함수는 간단한 추상화 방법이지만 한계가 있습니다. 함수는 상태를 가지고 있지 않으며, 함수의 시그니처를 변경하는 것은 대부분 모든 사용처에 영향을 미칩니다.
구현을 추상화하는 더 강력한 방법은 클래스를 사용하는 것입니다.

---

## 클래스를 이용한 추상화

메시지 표시 기능을 클래스로 추상화하는 방법은 아래와 같습니다.

```kotlin
class MessageDisplay(val context: Context) {
    fun show(
        message: String,
        length: MessageLength = MessageLength.LONG
    ) {
        val toastDuration = when (length) {
            SHORT -> Toast.LENGTH_SHORT
            LONG -> Toast.LENGTH_LONG
        }

        Toast.makeText(context, message, toastDuration).show()
    }
}

enum class MessageLength { SHORT, LONG }

val msgDisplay = MessageDisplay(context)
msgDisplay.show("Hello World!")
```

클래스가 함수보다 더 강력한 이유는 클래스는 상태를 유지할 수 있고 여러 함수를 노출할 수 있기 때문입니다. 
위의 예시처럼 클래스의 상태는 `context`이며, 이는 생성자를 통해 주입됩니다. 

아래와 같이 DI 프레임워크를 사용하면 클래스 생성을 위임할 수 있습니다.

```kotlin
@Inject lateinit var msgDisplay: MessageDisplay
```

또한, 테스트를 위해 해당 클래스에 의존하는 다른 클래스의 기능을 테스트하기 위해 클래스를 mockking 할 수 있습니다.

```kotlin
val msgDisplay: MessageDisplay = mockk()
```

추가적으로, 메시지 표시 설정을 위한 다양한 메서드를 추가할 수 있습니다.

```kotlin
msgDisplay.setChristmasMode(true)
```

이처럼, 클래스는 개발자에게 많은 유연성을 제공하지만 그럼에도 여전히 제한 사항이 있습니다.

클래스가 `final`로 선언된 경우 해당 클래스 타입 아래에 어떤 구현체가 존재하는지를 파악해야 합니다.
그렇기 떄문에 해당 클래스를 변경하거나 확장하는 것이 제한될 수 있습니다.

`open` 클래스를 사용하면 기본 클래스를 상속받은 서브 클래스를 통해 다양한 기능을 추가하거나 기존 기능을 수정하는 등의 확장을 통해 `final`에 대한 제한을 약간 완화할 수 있습니다.
그럼에도, 추상화는 여전히 기본 클래스에 강하게 연결되어 있으므로, 원본 클래스의 변경 없이 서브 클래스의 동작을 완전히 변경하는 것은 어려울 수 있습니다.

이를 해결하기 위해 클래스를 더 추상적인 인터페이스 뒤로 숨겨서 사용할 수 있습니다.

---

## 인터페이스 활용 추상화

Kotlin 표준 라이브러리를 살펴보면, 거의 모든 것이 인터페이스로 표현되는 것을 알 수 있습니다.

- `listOf()`는 `List` 인터페이스를 반환합니다. 이는 다른 팩토리 메서드들에게도 찾아 볼 수 있는 패턴입니다.
- 컬렉션 처리 함수들은 `Iterable`이나 `Collection`에 대한 확장 함수로 `List`, `Map` 등의 인터페이스를 반환합니다.
- 프로퍼티 델리게이트는 `ReadOnlyProperty`나 `ReadWriteProperty`라는 인터페이스 뒤에 숨겨져 있습니다.
- `lazy` 함수도 반환 타입으로 `Lazy`라는 인터페이스를 선언하고 있습니다.

라이브러리 개발자들은 보통 내부 클래스의 가시성(visibility)을 제한하고 인터페이스를 통해 이를 노출하는 방식을 택합니다.
이는 사용자들이 이 클래스들을 직접 사용하지 않으므로 인터페이스만 유지하면서 해당 구현을 자유롭게 변경할 수 있는 이점이 있기 때문입니다.

> 위와 같이 객체를 인터페이스 뒤에 숨김으로써 실제 구현을 추상화하고, 사용자가 이 추상화에만 의존하도록 강제함으로써 결합도를 낮추는것이 인터페이스를 활용한 추상화의 핵심입니다.

위 핵심내용을 토대로 클래스를 인터페이스 뒤로 숨기는 예시를 살펴보겠습니다.

```kotlin
interface MessageDisplay {
    fun show(message: String, duration: MessageLength = LONG)
}

class ToastDisplay(val context: Context): MessageDisplay {
    override fun show(message: String, duration: MessageLength) {
        val toastDuration = when (duration) {
            SHORT -> Toast.LENGTH_SHORT
            LONG -> Toast.LENGTH_LONG
        }

        Toast.makeText(context, message, toastDuration).show()
    }
}

enum class MessageLength { SHORT, LONG }
```

위와 같이 변경되면 태블릿에서는 토스트를 스마트폰에서는 스낵바를 표시하는 클래스를 주입하는 등의 더 많은 유연성을 얻을 수 있습니다.
또한 `MessageDisplay`는 Android, iOS, 웹 간에 공유되는 공통 모듈에서도 사용할 수 있습니다.

또 다른 이점은, 인터페이스의 가짜구현이 클래스를 모킹보다 간단하며, 모킹 라이브러리를 필요로 하지 않습니다.

```kotlin
val msgDisplay: MessageDisplay = TestMessageDisplay()
```

마지막으로 선언과 사용이 분리되어 있어, `ToastDisplay`와 같은 실제 클래스를 변경하는데 더 유연합니다.
그러나, 사용 방식을 변경하려면 `MessageDisplay` 인터페이스와 이를 구현하는 모든 클래스를 변경해야 하는것이 불편할 수 있습니다.

---

## 추상화 문제점 및 균형

추상화는 코드의 복잡성을 줄이고 재사용성을 높이는 동시에, 새로운 개념의 도입이 필요하며 프로젝트를 이해하는 데에 추가적인 학습 요소가 됩니다.   
이를 제한하는 방법은 추상화의 가시성을 제한하거나 특정 작업에만 사용되는 추상화를 정의하는 것입니다. 하지만 이러한 방법이 모든 문제를 해결하지는 않습니다.   
모든 것을 무작정 추상화하는 것보다는 모듈화의 중요성을 인지하고, 비용과 효과를 고려하여 적절한 균형을 찾아야 합니다.

추상화는 많은 것을 숨기고, 생각해야 할 것이 더 적어져 개발 작업이 간편해질 수 있지만, 과도한 추상화는 행동의 결과를 파악하거나 코드를 이해하는 것을 어렵게 만들 수 있습니다. 또한, 불확실한 결과에 대한 불안감을 유발할 수 있습니다.

추상화의 적절한 균형은 프로젝트의 복잡성, 팀의 규모와 경험, 프로젝트의 크기, 도메인 배경 지식 등 다양한 요소를 고려해야 합니다. 
이를 위해 필요한 직관은 수백, 수천 시간 동안의 프로젝트 설계와 코딩 경험을 통해 얻을 수 있습니다.

예를 들어, 대규모 프로젝트에서는 개발자 수가 많아 객체의 생성 및 사용 방법을 바꾸는 것이 어려울 수 있습니다. 
따라서 더 추상적인 솔루션을 선호하며 이 때 모듈 간의 분리가 특히 유용합니다.   
반면, 프로젝트 규모가 작거나 실험적일 때는, 추상화 없이도 직접 변화를 자유롭게 적용할 수 있습니다. 
하지만 프로젝트가 본격적으로 진행되면 가능한 빨리 변경하는 것이 좋습니다.

이외에도, DI 프레임워크를 사용할 때는 대부분 1번만 정의하면 되기 때문에 객체 생성의 복잡성에 대해 크게 걱정하지 않아도 되며, 테스트 수행 및 앱의 다양한 버전 생성 시 추상화를 활용할 수 있습니다.

결국, 적절한 추상화는 프로젝트의 특성, 팀의 역량, 그리고 시간에 따라 달라집니다. 
추상화의 문제점을 이해하고 이를 극복하는 방안을 찾는 것은 개발 프로세스의 중요한 부분입니다. 
이를 통해 개발 프로세스를 더 효과적으로 관리하고, 코드의 효율성과 가독성을 높일 수 있습니다.