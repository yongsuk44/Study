함수만 'inline'으로 처리될 수 있는 것이 아니다.  단일 값을 보유한 객체 또한 해당 값으로 대체될 수 있다.  
이를 사용하기 위해선 단일 프로퍼티를 가진 'primary constructor'의 클래스 앞에 'inline' modifier를 추가하면 된다. 

```kotlin
inline class Name(private val value: String) {
    // ...
}

// Code
val name: Name = Name("John")

// During compilation replaced with code similar to
val name: String = "John"
```

이러한 'inline class'의 메서드들은 'static method'로 정의된다.

```kotlin
inline class Name(private val value: String) {
    // ...
    
    fun greet() {
        print("Hello, I am $value")
    }
}

// Code
val name: Name = Name("John")
name.greet()

// During compilation replaced with code similar to
val name: String = "John"
Name.`greet-impl`(name)
```

위와 같이, 어떤 타입 주변에 wrapper를 만들 때, 성능 저하 없이 'inline class'를 활용할 수 있다.  
특히, 인기 있는 'inline class'를 사용 사례는 다음과 같다.

1. Indicate unit of measure
2. To use types to protect user from misuse

---

## Indicate unit of measure

타이머 설정을 위해 사용되는 메서드의 예시를 살펴보자.

```kotlin
interface Timer {
    fun callAfter(time: Int, callback: () -> Unit)
}
```

여기서 'time'이 정확히 어떤 시간 단위를 나타내는지 명시되어 있지 않다.  
이는 밀리초, 초, 분 등 다양한 시간일 수 있다. 즉, 측정 단위의 혼동은 심각한 문제를 야기할 수 있다.

개발자들이 측정 단위를 명시하는 일반적인 방법 중 하나는 파라미터 이름에 해당 단위를  포함시키는 것이다.

```kotlin
interface Timer {
    fun callAfter(timeMillis: Int, callback: () -> Unit)
}
```

파라미터 이름에 측정 단위를 포함하는 방식이 어느 정도 개선을 가져왔음에도 불구하고, 여전히 실수를 할 수 있는 가능성을 제거하지 못한다.  
특히, 함수를 사용할 때는 해당 함수의 프로퍼티 이름이 항상 보이지 않기에, 이 방법의 효과가 제한될 수 있다.  
더욱이, 함수아 어떤 타입을 반환할 때 이런식으로 타입을 명시하는 것은 더욱 어렵다.

예를 들어, 'decideAboutTime' 함수에서 시간을 반환하는 경우, 
그 시간의 단위가 전혀 명시되지 않았다면 함수가 분 단위 시간을 반환하고 있다면 시간 설정이 올바르게 작동하지 않을 위험이 있다.

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

'decideAboutTimeMillis'로 명명하여 반환되는 시간의 단위가 밀리초임을 나타내는 것처럼, 반환 값의 단위를 함수 이름에 포함시키는 방법도 있다.  
하지만, 이런 방식은 함수 이름을 매번 사용할때마다 더 길게 만들어야 하고, 필요하지 않은 상황에서도 저수준의 정보를 명시하게 되므로 드물게 사용된다.

이 문제를 해결하는 보다 나은 방법은 제네릭 타입의 잘못된 사용을 방지할 수 있는 엄격한 타입을 도입하는 것이며, 이를 효율적으로 만들기 위해 'inline class'를 사용할 수 있다.
이는 'inline class'가 런타임에 추가적인 객체 할당 없이 기본 타입 값으로 취급되기 때문에 성능 저하 없이 타입 안전성을 향상 시킬 수 있다. 

```kotlin
inline class Minutes(val minutes: Int) {
    fun toMillis(): Millis = Millis(minutes * 60 * 1000)
}

inline class Millis(val millisSeconds: Int) {
    // ...
}

interface User {
    fun decideAboutTime(): Minutes
    fun wakeUp()
}

interface Timer {
    fun callAfter(timeMillis: Millis, callback: () -> Unit)
}

fun setUpUserWakeUpUser(user: User, timer: Timer) {
    val time = user.decideAboutTime()
    timer.callAfter(time.toMillis()) {
        user.wakeUp()
    }
}
```

'metric' 단위를 다룰 때 위와 같은 방법이 특히 유용하다.  
프론트엔드 개발에서는 'pixel', 'millimeter', 'dp' 등의 단위를 다루는 경우가 많기 때문이다.  
또한, 객체 생성을 용이하게 하기 위해 DSL과 같은 확장 프로퍼티를 정의할 수 있다.

```kotlin
val Int.min get() = Minutes(this)
val Int.ms get() = Millis(this)

val timeMin: Minutes = 10.min
```

---

## Protect us from type misuse

SQL 데이터베이스 환경에서는 요소들을 숫자로 된 ID로 식별하는 거이 일반적이다.  
이러한 상황에서, 시스템 내 학생의 성적을 처리한다고 할 때, 이는 학생, 교서, 학교 등의 ID를 참조할 필요가 있을 것이다.

```kotlin
@Entity(tableName = "grades")
class Grades(
    @ColumnInfo(name = "studentId") val studentId: Int,
    @ColumnInfo(name = "teacherId") val teacherId: Int,
    @ColumnInfo(name = "schoolId") val schoolId: Int
    // ...
)
```

문제는 이러한 ID들을 잘못 사용하는 것이 매우 쉬워진다는 것이다.  
왜냐하면 이들 모두 'Int' 타입이기에, 타입 시스템이 잘못된 사용으로부터 보호해주지 않는다.  
이러한 문제를 해결하기 위한 방법은 모든 정수를 별도의 'inline class'로 래핑하는 것이다.

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

이제 각각의 ID 사용이 보다 안전해지며, 컴파일 과정에서 모든 타입들이 결국 'Int'로 치환되기에, 데이터베이스도 정확히 생성된다.  
'inline class'를 통해 이전에 타입을 사용할 수 없었던 영역에 타입을 도입할 수 있게 되었고, 이로 인해 성능 저하 없이 더 안전한 코드를 확보할 수 있게 된다.

---

## Inline classes and interfaces

'inline class'는 다른 종류의 클래스처럼 인터페이스를 구현할 수 있다.  
이러한 인터페이스를 활용하면, 다음과 같이 원하는 어떠한 측정 단위로든 시간을 정확하게 전달할 수 있다.

```kotlin
interface TimeUnit { 
    val millis: Long
}

inline class Minutes(val minutes: Long): TimeUnit {
    override val millis: Long
        get() = minutes * 60 * 1000
}

inline class Millis(val milliseconds: Long): TimeUnit {
    override val millis: Long 
        get() = milliseconds
}

fun setUpTimer(time: TimeUnit) {
    val millis = time.millis
    // ...
}

setUpTimer(Minutes(10))
setUpTimer(Millis(1000))
```

하지만, 객체가 인터페이스를 통해 사용될 때는 'inline' 처리 될 수 없다는 한계가 있다.  
이로 인해, 위에서 언급된 예처럼 'inline class'를 사용해도, 이 인터페이스를 통해 타입을 표현하기 위해서는 래핑된 객체를 생성해야 하므로 'inline class'의 장점을 활용할 수 없다.
인터페이스를 통해 'inline class'를 제시할 때, 해당 클래스들은 'inline' 처리 되지 않는다.

---

## Typealias

Kotlin의 'typealias'는 기존 타입에 대해 새로운 이름을 부여할 수 있다.

```kotlin
typealias NewName = Int
val n: NewName = 10 
```

길고 반복적인 타입을 다루는 상황에서 타입에 새로운 이름을 붙이는 것은 매우 유용하다.  
예를 들어, 반복되는 함수 타입에 이름을 붙이는 것은 널리 사용되는 방법 중 하나이다.

```kotlin
typealias ClickListener = (view: View, event: Event) -> Unit

class View {
    fun addClickListener(listener: ClickListener) {  }
    fun removeClickListener(listener: ClickListener) {  }
}
```

'typealias'은 기본적으로 타입에 새로운 이름을 부여하는 것이지, 타입 오용으로부터 보호해주지는 않는다.  
예를 들어, 'Int' 타입에 'Millis'와 'Seconds'라는 이름을 동시에 부여한다면, 타입 시스템이 보호해준다는 착각을 일으킬 수 있지만, 실제로 그렇지 않다.

```kotlin
typealias Seconds = Int
typealias Millis = Int

fun getTime(): Millis = 10
fun setUpTimer(time: Seconds) { }

fun main() {
    val seconds: Seconds = 10
    val millis: Millis = seconds // No Compiler Error
    
    setUpTimer(getTime())
}
```

위와 같이, 'typealias'를 사용하지 않는 경우가 오류를 찾기 더 쉬울 수 있기에, 위와 같은 방식을 활용하는 것이 적절하지 않음을 알 수 있다.  
측정 단위를 명시할 때는 변수 이름 또는 클래스를 활용하는 것이 좋다.   
변수 이름을 사용하는 방법이 비용 측면에서 유리할 수 있으나, 클래스를 사용하면 보다 높은 수준의 타입 안전성을 보장받을 수 있다.  

'inline class'를 사용하게 되면, 비용 효율성과 타입 안전성의 장점을 모두 누릴 수 있다.