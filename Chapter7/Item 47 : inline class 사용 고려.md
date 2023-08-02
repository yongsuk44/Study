# Item 47: inline class 사용 고려 

함수만 `inline`화 할 수 있는것이 아니며, 단일 값을 가진 객체도 이 값으로 대체될 수 있습니다.
이러한 기능을 가능하게 하려면 단일 생성자 속성이 있는 클래스 앞에 `inline` 수식어를 붙여야 합니다.

```kotlin
inline class Name(private val value: String) {
    // ...
}

val name: Name = Name("John")

// 컴파일 중 아래와 같이 유사하게 변경
val name: String = "John"
```

`inline` 클래스의 메서드들은 정적메서드로 정의됩니다.

```kotlin
inline class Name(private val value: String) {
    // ...
    fun greet() = print("Hello, I am $value")
}

val name: Name = Name("John")
name.greet()

// 컴파일 중 아래와 같이 유사하게 변경
val name: String = "John"
Name.`greet-impl`(name)
```

`inline class`를 통해 일부 유형(위 `String`과 같은)을 래핑할 수 있으며 성능 오버헤드 없이 사용할 수 있습니다.

`inline class`는 2가지 용도로 사용됩니다.
1. 측정 단위 나타내기 
2. 타입을 사용하여 사용자 오용 방지

---

## 측정 단위 표현

다음과 같이 타이머와 같이 측정 단위를 표현하는 상황을 예시로 들어 볼 수 있습니다.

```kotlin
interface Timer { 
    fun callAfter(timeMillis: Int, callback: () -> Unit) 
}
```

위 타이머 사용 시 파라미터 이름이 종종 보이지 않아 실수할 수 있습니다.
또한 이러한 방식으로 타입을 지정하면 반환 타입에 대해 지정하는 것이 어려울 수 있습니다.

아래 예시는 `decideAboutTime`에서 시간이 반환되지만 측정 단위가 명확하지 않습니다.

```kotlin
interface User {
    fun decideAboutTime(): Int
    fun wakeUp()
}

interface Timer {
    fun callAfter(timeMillis: Int, callback: () -> Unit)
}

fun setUpUserWakeUpUser(user: User, timer: Timer) {
    val time: Int = user.decideAboutTime()
    timer.callAfter(time) { 
        user.wakeUp()
    }
}
```

반환 값의 측정 단위를 함수 이름으로 지을 수 있지만, 이러한 해결책은 함수 이름 길이가 길어지고 굳이 필요하지 않기에 드물게 사용됩니다.

이러한 문제를 해결하는 더 좋은 방법은 `inline class`를 사용하여 이를 효율적으로 만드는 방법이 있습니다.

```kotlin
inline class Millis(val millisSeconds: Int) {
    // ...
}

inline class Minutes(val minutes: Int) {
    fun toMillis(): Millis = Millis(minutes * 60 * 1000)
}

interface User {
    fun decideAboutTime(): Minutes
    fun wakeUp()
}

interface Timer {
    fun callAfter(time: Millis, callback: () -> Unit)
}

fun setUpUserWakeUpUser(user: User, timer: Timer) {
    val time: Millis = user.decideAboutTime()
    timer.callAfter(time.toMillis()) { 
        user.wakeUp()
    }
}
```

특히 pixels, millimeters, dp 등 `metric` 단위의 경우에 유용하게 사용할 수 있습니다.
또한 객체 생성을 지원하기 위해 DSL과 같은 유사한 확장 속성을 정의할 수 있습니다.

```kotlin
val Int.min get() = Minutes(this)
val Int.ms get() = Millis(this)

val timeMin: Minutes = 10.min
```