# Item 40: equals 계약 준수

Kotlin에서 모든 객체는 `Any`를 상속합니다.

`Any`는 몇 가지 메서드와 그에 해당하는 계약(Contract)을 포함하고 있습니다.

- `equals`
- `hashCode`
- `toString`

위 메서드들은 Kotlin에서 중요한 역할을 하며, Java가 처음 생겨났을 때 부터 정의되어 왔기 때문에 많은 객체와 함수들이 이 메서드들의 계약에 의존하고 있습니다.
위 계약을 어기게 되면 일부 객체나 함수가 제대로 작동되지 않을 수 있습니다.

## 동등성(Equality)

Kotlin에서는 2가지 타입의 동등성이 존재합니다.

### 구조적 동등성(Structural Equality)

`equals` 메서드나 `==` 연산자를 통해 확인할 수 있습니다. 이는 `equals` 메서드에 기반을 두고 있습니다.

`a == b`는 `a != null`이면 `a.equals(b)`로 변환되며, 그렇지 않으면 `a?.equals(b) ?: (b === null)`로 변환됩니다.

### 참조 동등성(Referential Equality)

`===` 연산자를 통해 확인할 수 있으며 두 측이 같은 객체를 가리킬 때 `true`를 반환합니다.

`equals`는 모든 클래스의 `super class`인 `Any`에서 구현되므로, 어떤 두 객체의 동등성도 체크할 수 있습니다.

```kotlin
open class Animal
class Book
class Cat : Animal()

Animal() == Book() // Error : Operator == cannot be applied to Animal and Book
Animal() === Book() // Error : Operator === cannot be applied to Animal and Book

Animal() == Cat() // OK, because Cat is a subclass of Animal
Animal() === Cat() // OK, because Cat is a subclass of Animal
```

---

## equals 존재 이유

`Any`로부터 상속받은 기본 `equals` 구현은 다른 객체가 정확하게 같은 인스턴스인지 확인합니다.

이는 참조 동등성(`===`)과 동일하며, 기본적으로 모든 객체가 고유하다는 것을 의미하기도 합니다.

```kotlin
class Name(val name: String)

val name1 = Name("John")
val name2 = Name("John")
val name1Ref = name1

name1 == name1 // true
name1 == name2 // false
name1 == name1Ref // true

name1 === name1 // true
name1 === name2 // false
name1 === name1Ref // true
```

여러 객체에서는 위와 같은 동작이 유용하며 특히 DB연결, 레포지토리, 스레드와 같은 활성화된 요소에 적합합니다.

그러나 동등성을 다르게 표현해야하는 객체도 있습니다. 모든 기본 생성자 속성이 같은지 확인하는 데이터 클래스 동등성이 대표적인 예시 입니다.

```kotlin
data class FullName(val name: String, val surname: String)

val fullName1 = FullName("John", "Smith")
val fullName2 = FullName("John", "Smith")
val fullName3 = FullName("John", "Doe")

fullName1 == fullName1 // true
fullName1 == fullName2 // true, because data are the same
fullName1 == fullName3 // false

fullName1 === fullName1 // true
fullName1 === fullName2 // false
fullName1 === fullName3 // false
```

위와 같은 방식은 데이터에 의해 정의되는 클래스에 적합하며 이러한 이유로 데이터 모델 클래스나 데이터 홀더에 주로 data modifier를 사용합니다.

다른 불필요한 속성을 건너뛸 필요가 있을 때 데이터 클래스 동등성은 일부 속성만 비교할 때도 도움이 됩니다.

아래 예시는 `asStringCache`와 `changed` 속성을 비교에서 제외해야 하는 날짜와 시간을 나타내는 객체입니다.

```kotlin
class DateTime(
    private var millis: Long = 0L,
    private var timeZone: TimeZone? = null
) {
    private var asStringCache: String = ""
    private var changed: Boolean = false

    override fun equals(other: Any?): Boolean =
        other is DateTime &&
                millis == other.millis &&
                timeZone == other.timeZone
}
```

위 예시는 아래에서와 같이 data modifier 만을 추가하면 동일한 결과를 얻을 수 있습니다.

```kotlin
data class DateTime(
    private var millis: Long = 0L,
    private var timeZone: TimeZone? = null
) {
    private var asStringCache: String = ""
    private var changed: Boolean = false

    // ...
}
```

그러나, 이 경우에는 복사 동작이 기본 생성자에 선언되지 않은 속성은 복사하지 않습니다. 이는 추가적인 속성들이 중요하지 않을 때만 올바른 동작입니다.

기본 동등성과 데이터 클래스 동등성이라는 이 두 가지 대안을 통해, Kotlin에서는 거의 직접 동등성을 구현해야 할 필요가 없습니다.

그럼에도 불구하고, 직접 `equals`를 구현해야하는 경우가 있습니다.

```kotlin
class User(
    val id: Int,
    val name: String
) {
    override fun equals(other: Any?): Boolean =
        other is User && id == other.id

    override fun hashCode(): Int = id
}
```

아래와 같은 경우에는 `equals`를 직접 구현해야 합니다.

- 동등성 판단 로직이 기본적인 것과 다르게 필요한 경우
- 속성 중 일부만 비교하고 싶을 때
- 객체가 데이터 클래스가 되기를 원하지 않거나 비교해야 하는 속성들이 기본 생성자에 없는 경우

---

## equals 구현 요구 사항

`equals`메서드는 객체가 다른 객체와 '동일한지'를 나타내며 `equals`의 구현은 다음 기본 요구사항을 만족해야 합니다.

- [반사성(Reflexive)](#반사성) : 모든 `null`이 아닌 값 `x`에 대해, `x.equals(x)`는 `true`를 반환해야 합니다.
- [대칭성(Symmetric)](#대칭성) : 모든 `null`이 아닌 값 `x`와 `y`에 대해, `x.equals(y)`가 `y.equals(x)`가 `true`일 때만 `true`를 반환해야 합니다.
- [전이성(Transitive)](#전이성) : 모든 `null`이 아닌 값 `x`, `y`, `z`에 대해, `x.equals(y)`가 `true`를 반환하고 `y.equals(z)`가 `true`를 반환하는
  경우, `x.equals(z)` 역시 `true`를 반환해야 합니다.
- [일관성(Consistent)](#일관성) : 모든 `null`이 아닌 값 `x`와 `y`에 대해, `equals` 비교에 사용되는 객체 정보가 수정되지 않는 한 `x.equals(y)`를 여러 번 호출하더라도
  일관성있게 `true`를 반환하거나 일관성있게 `false`를 반환해야 합니다.
- `null`과 절대 동일하지 않음 : 모든 `null`이 아닌 값 `x`에 대해, `x.equals(null)`은 항상 `false`를 반환해야 합니다.

추가적으로, 공식적인 계약(contract)의 일부는 아니지만, `equals`, `toString`, `hashCode` 메서드는 빠르게 동작해야 합니다.  
만약 두 객체가 동일한지 확인하는 데 몇 초를 기다리는 것은 좋지 않은 사용성을 제공할 수 있습니다.

### 반사성

객체 동등성은 반사성을 가져야 합니다. 즉, `x.equals(x)`는 항상 `true`를 반환해야 합니다.
이 조건은 당연한 것처럼 보일 수 있지만, 위반하는 경우가 있습니다.

아래는 현재 시간을 표현하는 `Time` 객체를 만들어서 밀리초를 비교하고자 하는 상황입니다.

```kotlin
class Time(
    val millisArg: Long = -1,
    val isNow: Boolean = false
) {
    val millis: Long
        get() =
            if (isNow) System.currentTimeMillis() else millisArg

    override fun equals(other: Any?): Boolean =
        other is Time && millis == other.millis
}

val now = Time(isNow = true)
now == now // 때때로 `true`, 때때로 `false`를 반환
List(100000) { now }.all { it == now } // 거의 확실히 false를 반환
```

객체가 자신과 동일하지 않다면 대부분의 컬렉션에서는 그 객체를 찾지 못합니다.
즉, `contains` 메서드를 사용해 확인하여도 그 객체가 존재하지 않는 것처럼 나올 수 있습니다.

대부분의 Unit Test `Assert`에서도 통과되지 않을 것입니다.

```kotlin
val now1 = Time(isNow = true)
val now2 = Time(isNow = true)

assertEquals(now1, now2) // 때때로 통과, 때때로 실패
```

이처럼 이 결과가 올바른 것인지, 일관성이 없기 때문인지 확실하게 알 수 없기에 신뢰할 수 없게 됩니다.

위 문제를 해결하기 위해서는 객체가 현재 시간을 나타내는지 확인하고, 그렇지 않은 경우엔 동일한 타임스탬프를 가지는지 확인하는 것 입니다.
그러나, 위 해결책은 태그 클래스의 전형적인 사례로써 클래스 계층 구조를 사용하는 것이 더 좋은 방법일 수 있습니다.

```kotlin
sealed class Time
data class TimePoint(val millis: Long) : Time()
object Now : Time()
```

### 대칭성

객체 동등성은 대칭성을 가져야 합니다. 즉, `x == y` 이라면 `y == x`도 반드시 성립해야 합니다.
이 원칙은 다른 유형의 객체를 동등성 비교에 포함할 때 위반될 수 있습니다.

아래는 복소수를 나타내는 클래스에서 `Double`을 동등성 비교에 허용하는 예시입니다.

```kotlin
class Complex(
    val real: Double,
    val imaginary: Double
) {
    // 이렇게 할 경우, 대칭성 위반 
    override fun equals(other: Any?): Boolean {
        if (other is Double) {
            return imaginary == 0.0 && real == other
        }

        return other is Complex &&
                real == other.real &&
                imaginary == other.imaginary
    }
}
```

위 문제는 `Double`이 `Complex`와의 동등성이 인정되지 않다는 것입니다.

그 결과, 비교하는 두 요소의 순서에 따라 결과가 달라집니다.

```kotlin
Complex(1.0, 0.0).equals(1.0) // true
1.0.equals(Complex(1.0, 0.0)) // false
```

위처럼 대칭성이 위반되는 경우 컬렉션의 `contains`나 UnitTest의 `Assert`에서 예상치 못한 결과를 가져올 수 있습니다.

```kotlin
val list = listOf<Any>(Complex(1.0, 0.0))
list.contains(1.0) // 현재 JVM에서는 false를 반환하지만, 이는 컬렉션의 구현에 따라 달라질 수 있음
```

이처럼 대칭성이 위반되는 경우, 예상치 못한 오류를 발생시키고 디버깅이 어려워질 수 있습니다.
이로 인해 `equals`를 구현할 때는 항상 동등성을 고려해야 합니다.

일반적인 해결책은 다른 클래스 간의 동등성을 허용하지 않는 것입니다.
Kotlin에서는 비슷한 유형의 클래스들이 동등하지 않습니다. 예를 들어, 1은 1.0과 같지 않으며, 1.0은 1.0F와 같지 않습니다. 이들은 각각 다른 유형이며 비교가 불가능합니다.
또한, Kotlin에서는 `Any`를 제외한 공통 상위 클래스가 없는 두 가지 다른 유형 간에 `==` 연산자를 사용할 수 없습니다.

```kotlin
`Complex(1.0, 0.0) == 1.0` // 에러 발생
```

### 전이성

객체 동등성은 전이성을 가져야 합니다. 즉, `x == y` 이고 `y == z`이면 `x == z`도 반드시 성립해야 합니다.  
전이성을 해치는 가장 큰 문제는 서로 다른 서브타입의 속성을 비교하는 동등성 구현을 만들 때 발생합니다.

아래는 Date와 DateTime을 정의한 예시 입니다.

```kotlin
open class Date(
    val year: Int,
    val month: Int,
    val day: Int
) {
    // 이렇게 작성되면 대칭성은 지켜지지만 전이성이 위반됨
    override fun equals(other: Any?): Boolean =
        when (other) {
            is DateTime -> this == other.date
            is Date -> year == other.year &&
                    month == other.month &&
                    day == other.day
            else -> false
        }

    // ...
}

class DateTime(
    val date: Date,
    val hour: Int,
    val minute: Int,
    val second: Int
) {
    // 이렇게 작성되면 대칭성은 지켜지지만 전이성이 위반됨
    override fun equals(other: Any?): Boolean =
        when (other) {
            is DateTime -> date == other.date &&
                    hour == other.hour &&
                    minute == other.minute &&
                    second == other.second
            is Date -> date == other
            else -> false
        }

    // ...
}
```

위 구현에서 발생되는 문제점은 두 `DateTime` 객체 비교 시 `Date`와 `DateTime`을 비교하는 것보다 더 많은 속성을 고려한다는 점입니다.
따라서, 같은 날짜이지만 시간이 다른 두 `DateTime` 객체는 서로 같지 않지만, 같은 `Date` 객체와는 같을 수 있습니다.

결과적으로 이들은 전이성을 위반하고 있습니다.

```kotlin
val o1 = DateTime(Date(1992, 10, 20), 12, 30, 0)
val o2 = Date(1992, 10, 20)
val o3 = DateTime(Date(1992, 10, 20), 14, 45, 30)

o1 == o2 // true
o2 == o3 // true
o1 == o3 // false <- 전이성 위반
```

위와 같은 경우, 동일한 유형의 객체만 비교하는 제한은 도움이 되지 않습니다. 이는 상속을 사용해 리스코프 치환 원칙을 위반하기 때문입니다.

이럴 때는 상속 대신 구성을 사용하는 것이 바람직합니다. 이렇게 하면 다른 두 유형의 객체를 비교하는 상황을 회피할 수 있습니다.
데이터를 가지는 객체 표현 시 아래와 같은 구현이 좋은 방법입니다.

```kotlin
data class Date(
    val year: Int,
    val month: Int,
    val day: Int
)

data class DateTime(
    val date: Date,
    val hour: Int,
    val minute: Int,
    val second: Int
)

val o1 = DateTime(Date(1992, 10, 20), 12, 30, 0)
val o2 = Date(1992, 10, 20)
val o3 = DateTime(Date(1992, 10, 20), 14, 45, 30)

o1.equals(o2) // false
o2.equals(o3) // false
o1 == o3 // false

o1.date.equals(o2) // true
o2.equals(o3.date) // true
o1.date == o3.date // true
```

### 일관성

객체 동등성은 항상 일관성이 있어야 합니다. 즉, 같은 두 객체에 대해 호출된 `equals`는 항상 동일한 결과를 반환해야 합니다. 불변 객체의 경우, 결과는 항상 일정해야 합니다.   
즉, `equals`는 순수 함수(pure function)이어야 하며, 그 결과는 항상 입력값과 수신자 객체의 상태에만 의존해야 합니다.

위 반사성에서 `Time` 클래스에서 이 원칙을 위반하는 예를 보았으며, 이 규칙은 자바의 `java.net.URL.equals()`에서도 유명하게 위반되었습니다.

### not-null x : x.equals(null) == false

모든 not-null 값 `x`에 대해 `x.equals(null)`은 항상 `false`를 반환해야 합니다.
`null`은 고유해야 하며, 어떤 객체도 `null`과 동등하다고 볼 수 없기 때문입니다.

---

## URL equals()의 문제점

`java.net.URL`의 객체의 동등성이 네트워크 작업에 의해 결정되는 설계 오류를 가지고 있습니다.

두 호스트가 동등하다고 판단되는 경우는 두 호스트 이름이 같은 IP 주소로 해석될 수 있을 때입니다.
그러나, 동등성은 절대로 네트워크에 의존해서는 안됩니다.

```kotlin
import java.net.URL

fun main() {
    val enWiki = URL("http://en.wikipedia.org/")
    val enWiki2 = URL("http://en.wikipedia.org/")
  
    println(enWiki == enWiki2) // 인터넷이 연결되어 있을 때는 true, 그렇지 않으면 false
}
```

위 방법의 문제점은 다음과 같습니다. 

### 위 동작은 일관성이 없음

두 `URL`은 네트워크가 사용 가능한 상태에서는 같은 것으로 여겨지지만, 네트워크가 사용 불가능한 상태에서는 같지 않을 수 있습니다.
또한 네트워크는 항상 변하며 특정 호스트 이름에 대한 IP 주소는 시간이 지나거나 네트워크가 변경됨에 따라 달라질 수 있습니다.
따라서 두 `URL`은 어떤 네트워크에서는 같은 것으로 여겨지고, 다른 네트워크에서는 그렇지 않을 수 있습니다.

### 네트워크 작업이 느릴 수 있음

`equals`와 `hashCode` 메서드는 빠르게 동작되는 것을 기대하지만 네트워크 작업이 느릴 수 있습니다.
이 둘은 컬렉션 등의 데이터 구조에서 주로 사용되며, 그 과정에서 네트워크 호출이 발생한다면 큰 성능 문제를 야기할 수 있습니다.

예를 들어 `URL`이 어떤 리스트에 포함되어 있는지 확인하는 경우, 리스트의 모든 요소에 대해 네트워크 호출이 발생할 수 있습니다.
Android와 같은 일부 플랫폼에서는 메인 스레드에서 네트워크 작업을 할 수 없기에, `URL` 집합에 추가하는 작업조차 별도의 스레드에서 수행해야 할 수 있습니다.

### 정의된 동작은 HTTP의 가상 호스팅과 일치하지 않음

동일한 IP 주소가 동일한 내용을 가진다는 보장이 없습니다.
가상 호스팅을 사용하면 서로 무관한 여러 사이트가 같은 IP 주소를 공유할 수 있습니다.
따라서 이 메서드는 서로 관련이 없는 두 `URL`을 동일하다고 판단할 수 있습니다.

Android는 이 문제를 Android 4.0에서 수정했으며 4.0 이후에는 호스트 이름이 동일할 경우에만 URL이 동등하다고 판단합니다.
다른 플랫폼에서 Kotlin/JVM을 사용할 경우에는 `java.net.URL` 대신 `java.net.URI`를 사용하는 것이 권장됩니다.