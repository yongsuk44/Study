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

---

## 타입 오용으로부터 보호

SQL DB에서 종종 ID로 요소를 식별하며 이 ID들은 모두 단순한 숫자인 경우가 많으며, 시스템에서 학생 성적이 있다고 가정하면 학생, 교사, 학교의 ID를 참조해야 할 가능성이 높습니다.

```kotlin
@Entity(tableName = "grades")
class Grades(
    @ColumnInfo(name = "studentId") val studentId: Int,
    @ColumnInfo(name = "teacherId") val teacherId: Int,
    @ColumnInfo(name = "schoolId") val schoolId: Int
    // ...
)
```

문제점으로 나중에 이러한 ID를 잘못 사용하기가 정말 쉽기에 모든 정수를 별도의 `inline class`로 래핑하는 것이 안전합니다.

```kotlin
inline class StudentId(val studentId: Int)
inline class TeacherId(val teacherId: Int)
inline class SchoolId(val schoolId: Int)

class Grades(
    @ColumnInfo(name = "studentId") val studentId: StudentId,
    @ColumnInfo(name = "teacherId") val teacherId: TeacherId,
    @ColumnInfo(name = "schoolId") val schoolId: SchoolId
    // ...
)
```

위와 같이 `inline class`를 통해 ID를 사용하는 것은 안전할 것이며 또한 컴파일 중 모든 타입이 `Int`로 대체되므로 DB가 올바르게 생성됩니다.
이렇게 하면 `inline class`를 통해 이전에 허용되지 않았던 위치에 타입을 도입할 수 있기에 성능 오버헤드 없이 더 안전한 코드를 가지게 됩니다.

---

## inline class와 interface

`inline class`는 다른 클래스와 동일하게 인터페이스를 구현할 수 있습니다. 이 인터페이스는 원하는 측정 단위를 제대로 전달해줄 수 있습니다.

```kotlin
interface TimeUnit { 
    val millis: Long
}

inline class Minutes(val minutes: Long): TimeUnit {
    override val millis: Long
        get() = minutes * 60 * 1000
}

inline class Millis(val milliseconds: Long): TimeUnit {
    override val millis: Long get() = milliseconds
}

fun setUpTimer(time: TimeUnit) {
    val millis = time.millis
}

setUpTimer(Minutes(10))
setUpTimer(Millis(1000))
```

`inline class`의 특징 중 하나는 객체가 `inline` 되어 성능 최적화가 가능하다는 것입니다.
그러나 인터페이스를 통해 객체를 사용하는 경우 이러한 `inline` 기능을 통한 성능 최적화를 할 수 없습니다.

`inline class`는 컴파일 시점에서 단일 값으로 대체되는 클래스로 불필요한 객체 생성이 없어 성능이 향상됩니다.
인터페이스는 특정 메서드 계약을 정의하며, `inline class`가 인터페이스를 구현하면 해당 인터페이스 타입을 통해 접근해야 하므로 실제 객체의 래핑이 필요합니다.
`inline class`가 인터페이스를 통해 접근되면 `inline class`의 인스턴스는 실제 객체로 존재해야 하므로 인라인 최적화의 이점을 잃게 됩니다.

위 예시에서 `TimeUnit` 인터페이스를 통해 `Minutes`와 `Millis` `inline class`를 사용하면, `inline class`의 인스턴스는 실제 객체로 작동해야 하므로 성능 이점을 얻을 수 없게 됩니다.
이에 따라 `inline class`가 인터페이스를 구현하는 경우 성능 최적화의 이점을 누리기 어려우며, 성능에 대한 고려 없이 타입 안정성만을 목표로 하는 경우에는 유용할 수 있습니다.
