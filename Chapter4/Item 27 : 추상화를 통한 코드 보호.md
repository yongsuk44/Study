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