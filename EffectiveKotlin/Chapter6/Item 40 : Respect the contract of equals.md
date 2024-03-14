Kotlin의 모든 객체는 'Any' 클래스를 확장하며, 'Any'는 몇 가지 중요한 메서드와 이에 따른 명확한 'contract'를 가진다.

- equals
- hashCode
- toString

위 메서드들의 'contract'는 주석에 설명되어 있으며 공식 문서에도 자세하게 설명되어 있다.  
'Item 32'에서 언급했듯이, 설정된 'contract'를 가진 타입의 하위 타입은 이 'contract'를 준수해야 한다.  
이 메서드들은 Kotlin, 그리고 그 기반이 되는 Java에서부터 오랫동안 정의되어 왔기에, 많은 객체와 함수가 이들의 'contract'에 의존하고 있다.  
만약, 이 'contract'를 위반하면, 일부 객체나 함수가 제대로 동작하지 않을 수 있다.  
이러한 이유로, 이 메서드들을 'overriding'하고 이들의 'contract'를 살펴보자.

## Equality

Kotlin에서는 두 타입의 'equality'가 존재한다.

- Structural equality
  - 'equals' 메서드나 '==' 연산자(그리고 부정 형태인 '!=')를 통해 확인하며, 객체의 내용이 서로 같은지를 비교한다. 
  - 'a == b'는 'a'가 'null'이 아닐 경우 'a.equals(b)'로 해석되며, 'null'일 경우 'a?.equals(b) ?: (b === null)'로 해석된다.
- Referential equality
  - '===' 연산자(그리고 부정 형태인 '!==')로 확인하며, 두 객체가 메모리 상에서 같은 위치를 가리키고 있을 때 'true'를 반환한다.
  - 즉, 두 변수가 정확히 동일한 객체를 참조하고 있을 경우에만 'true'를 반환한다.

'equals' 메서드는 모든 Kotlin 클래스의 최상위 클래스인 'Any'에 구현되어 있기에, 어떠한 두 객체 간의 동등함 여부도 확인할 수 있다.  
하지만, 객체들이 같은 타입이 아닌 경우에는 연산자를 이용한 동등성 검사가 불가능하다.

```kotlin
open class Animal
class Book

Animal() == Book()      // Error : Operator == cannot be applied to Animal and Book
Animal() === Book()     // Error : Operator === cannot be applied to Animal and Book
```

두 객체가 다른 타입일 때의 동등성을 검사하는 것은 의미가 없기 때문에,
두 객체가 동일한 타입을 갖거나, 하나가 다른 하나의 하위 타입일 때 동등성 검사를 할 수 있다.

```kotlin
class Cat : Animal()
Animal() == Cat()       // OK, because Cat is a subclass of Animal
Animal() === Cat()      // OK, because Cat is a subclass of Animal
```

---

## Why do we need equals?

'Any' 클래스에서 제공되는 'equals'의 기본 구현은, 주어진 다른 객체가 현재 객체와 동일한 인스턴스인지 확인한다.  
이는 'referential equality(===)'를 확인하는 것과 같다. 따라서, 기본적으로 모든 객체는 고유하다는 것을 의미한다.

```kotlin
class Name(val name: String)

val name1 = Name("John")
val name2 = Name("John")
val name1Ref = name1

name1 == name1      // true
name1 == name2      // false
name1 == name1Ref   // true

name1 === name1     // true
name1 === name2     // false
name1 === name1Ref  // true
```

이러한 동작은 많은 객체에게 유용하며, 특히 'DB 연결', '저장소', '스레드'와 같은 'active elements'에게 적합하다.  
하지만, 때때로 'equality'를 다르게 정의해야 할 필요가 있는 객체들도 있다.  
이 경우에 인기 있는 방법 중 하나는 'data class'의 'equality'를 사용하는 것이며, 이는 모든 'primary constructor'의 프로퍼티들이 서로 같은지 확인한다.

```kotlin
data class FullName(val name: String, val surname: String)

val fullName1 = FullName("John", "Smith")
val fullName2 = FullName("John", "Smith")
val fullName3 = FullName("John", "Doe")

fullName1 == fullName1      // true
fullName1 == fullName2      // true, because data are the same
fullName1 == fullName3      // false

fullName1 === fullName1     // true
fullName1 === fullName2     // false
fullName1 === fullName3     // false
```

이러한 방식은 데이터를 통해 표현되는 클래스에 적합하다.   
이에 따라 'data model class' 또는 'other data holder class'에서 'data' modifier를 자주 사용하곤 한다.

'data class'의 'equality'는 일부 프로퍼티를 비교하되, 모든 프로퍼티를 비교하지 않아야 할 때도 도움이 된다.  
예를 들어, 캐시나 불필요한 중복 프로퍼티 등의 'equality check'가 필요 없는 경우에 유용하다.  

아래는 날짜와 시간을 나타내는 객체에서 'asStringCache'와 'changed' 같은 프로퍼티들의 'equality check'를 하지 않도록 하는 예시이다. 

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

// The same can be achieved by using data modifier
data class DateTime(
  private var millis: Long = 0L,
  private var timeZone: TimeZone? = null
) {
  private var asStringCache: String = ""
  private var changed: Boolean = false

  // ...
}
```

이 경우, 'copy'를 사용할 때 'primary constructor'에 명시되지 않은 프로퍼티들은 복사되지 않는다는 점을 유의해야 한다.  
이러한 동작은 추가적인 프로퍼티들이 실제로 불필요하고, 해당 프로퍼티들 없이도 객체가 정상적으로 동작할 수 있을 때만 적절하다.

'기본적인 equality'와 'data class'를 이용한 'equality' 덕분에, Kotlin에서는 대부분의 경우에 별도로 'equality'를 구현할 필요가 없다.  
하지만, 특정 상황에서는 직접 'equals'를 구현해야 할 필요가 있다.

예를 들어, 두 객체가 동등한지를 결정하는 것이 특성 프로퍼티에 의존하는 경우가 있다.  
아래 'User' 클래스는 두 사용자의 'id'가 같을 때 두 사용자가 동등하다고 가정할 수 있다.

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

위와 같이 'equals'를 직접 구현해야하는 상황은 다음과 같다.

- 기본적으로 제공되는 'equals' 로직이 요구사항과 맞지 않을 때
- 객체의 모든 프로퍼티가 아닌, 일부 프로퍼티만을 비교해야 할 필요가 있을 때
- 해당 객체를 'data class'로 정의하고 싶지 않거나, 비교해야 할 특정 프로퍼티들이 'primary constructor'에 없는 경우

---

## The contract of equals

'equals' 메서드는 객체가 다른 객체와 'equal to' 한지를 나타낸다.  
이를 구현할 때는 다음과 같은 요구사항을 충족해야 한다.

- Reflexive : 'null'이 아닌 어떤 값 'x'에 대해, 'x.equals(x)'는 항상 'true'여야 한다.
- Symmetric : 'null'이 아닌 어떤 값 'x'와 'y'에 대해, 'x.equals(y)'가 'true'일 때에만 'y.equals(x)' 또한 'true'여야 한다.
- Transitive : 'null'이 아닌 어떤 값 'x', 'y', 'z'에 대해, 'x.equals(y)'와 'y.equals(z)'가 'true'이면, 'x.equals(z)'도 'true'여야 한다.
- Consistent : 'null'이 아닌 어떤 값 'x'와 'y'에 대해, 'equals' 비교에 사용되는 정보에 변동이 없다면, 'x.equals(y)'는 여러 번 호출되어도 일관되게 'true'를 반환하거나 'false'를 반환해야 한다.
- Never equal to null : 'null'이 아닌 어떤 값 'x'에 대해, 'x.equals(null)'은 항상 'false'여야 한다. 

또한, 공식적인 'contract'에 포함된 사항은 아니지만, 'equals', 'toString', 'hashCode' 메서드는 빠르게 동작해야 한다.  
만약, 두 객체가 같은지 비교하는데 몇 초 이상을 기다리는 것은 일반적으로 예상되지 않는 상황으로 간주된다.

---

객체의 'equality'는 '반사적(reflexive)'이어야 한다.   
즉, 'x.equals(x)'는 항상 'true'여야 한다.

이는 분명하고 당연한 규칙처럼 보이지만, 경우에 따라 위반될 수 있다.  
예를 들어, 현재 시간을 표현하기 위해 만든 'Time' 객체가 있을 때, 이 객체가 'ms' 단위까지 비교하려는 시도가 이 원칙을 위반할 수 있다.

```kotlin
// Don't do this
class Time(
    val millisArg: Long = -1,
    val isNow: Boolean = false
) {
    val millis: Long get() = if (isNow) System.currentTimeMillis() else millisArg

    override fun equals(other: Any?): Boolean =
        other is Time && millis == other.millis
}

val now = Time(isNow = true)
now == now                              // Sometimes true, sometimes false
List(100000) { now }.all { it == now }  // Most likely false
```

위와 같이 결과가 일관되지 않으므로, 이는 'equality'의 마지막 원칙에도 어긋나게 된다.  
만약 어떤 객체가 자신과 같다고 판단하지 않는다면, 해당 객체는 'contains' 메서드를 통해 확인할 때 실제로 그 컬렉션 안에 존재함에도 불구하고 발견되지 않을 가능성이 크다.  

이는 다음과 같이 대부분의 단위 테스트에서의 'assertion' 검사에도 제대로 동작하지 않을 것이다.

```kotlin
val now1 = Time(isNow = true)
val now2 = Time(isNow = true)

assertEquals(now1, now2)        // Sometimes passes, sometimes fails
```

이처럼 결과의 일관성이 결여되면, 결과가 정확하게 나온 것인지, 아니면 일관성 없는 동작 때문에 나온 것인지 확실히 알 수 없게 되어 그 결과에 대한 신뢰를 잃게 된다.

이를 개선하는 방법 중 하나는, 먼저 객체가 현재 시간을 표현하는지 확인한 뒤, 그렇지 않다면 해당 객체가 같은 'TimeStamp'를 가지고 있는지를 체크하는 것이다.  
이 방식은 '태그가 있는 클래스'의 전형적인 예시로, 'Item 39'에서 설명된 대로, 클래스 계층 구조를 사용하는 편이 훨씬 바람직하다.

```kotlin
sealed class Time {
    data class TimePoint(val millis: Long) : Time()
    data object Now : Time()
}
```

---

객체 간의 'equality'는 '대칭적(Symmetric)'이어야 한다. 즉, 'x == y'와 'y == x'의 결과는 항상 같아야 한다.  
하지만, 'equality check'에서 다른 타입의 객체를 허용하는 경우 이 원칙이 쉽게 위반될 수 있다.

예를 들어, 복소수를 나타내는 클래스를 만들고, 'Double'을 허용하는 'equality check'를 구현하는 경우를 살펴보자.

```kotlin
class Complex(
    val real: Double,
    val imaginary: Double
) {
    // Don't do this, violates symmetry 
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

위 문제는 'Double' 타입이 'Complex' 타입과 동등하다고 인정하지 않는데 있다.  
이로 인해, 비교의 결과가 비교하는 요소들의 순서에 영향을 받게 된다.

```kotlin
Complex(1.0, 0.0).equals(1.0)   // true
1.0.equals(Complex(1.0, 0.0))   // false
```

'Symmetric'의 부재는 컬렉션에서 'contains' 메서드를 사용할 때나 단위 테스트에서의 'assertion' 검사 시 예상 밖의 결과가 발생할 수 있음을 의미한다.

```kotlin
val list = listOf<Any>(Complex(1.0, 0.0))
list.contains(1.0)  // 현재 JVM에서는 false를 반환하지만, 이는 컬렉션의 구현에 따라 달라질 수 있음
```

'Symmetric'이 없는 'equality'는 신뢰할 수 없는 결과를 초래할 수 있다.  
만약, 한 객체가 'x'와 'y'를 비교하는 방식과 'y'와 'x'를 비교하는 방식이 다르다면, 결과는 이 비교의 순서에 따라 달라질 수 있다.  
이러한 사실은 공식 문서에 명시되어 있지 않으며, 대부분의 객체 생성자는 두 방식이 동일하게 동작할 것이라고 가정한다.  
하지만, 리팩토링 과정에서 비교 순서가 언제든지 변경될 수 있으며, 이는 대칭성이 없는 경우 예상치 못한 오류를 발생시켜 디버깅을 매우 어렵게 만들 수 있다.  
그래서 'equals' 메서드를 구현할 때는 항상 'equality'를 심도 있게 고려해야 한다.

일반적인 해결책은 서로 다른 클래스 간의 'equality'를 허용하지 않는 것이다.  
Kotlin에서는 서로 유사해 보이는 클래스라도 자동으로 동등하다고 간주하지 않는다.  
예를 들어, 1은 1.0, 그리고 1.0과 1.0F은 각각 다른 타입이며, 서로 비교가 불가능하다.  
더욱이, Kotlin에서는 'Any'를 제외한 공통된 상위 클래스가 없는 다른 두 타입 간에 '==' 연산자를 사용 할 수 없다.

```kotlin
Complex(1.0, 0.0) == 1.0 // Error
```

---

객체 간의 'equality check'는 '전이성(Transitive)'를 가져야 한다. 즉, 'x == y'와 'y == z'가 'true'일 때, 'x == z'도 'true'여야 한다.  
'Transitive'과 관련된 가장 큰 문제는 다른 하위 타입의 프로퍼티를 검사하기 때문에 서로 다른 종류의 'equality'를 구현할 때 발생한다.

예를 들어, 다음과 같이 정의된 'Date' 및 'DateTime'이 있다고 가정해보자.

```kotlin
open class Date(
    val year: Int,
    val month: Int,
    val day: Int
) {
    
    // Don't do this, symmetric but not transitive
    override fun equals(other: Any?): Boolean =
        when (other) {
            is DateTime -> this == other.date
            is Date -> (year == other.year) && (month == other.month) && (day == other.day)
            else -> false
        }

    // ...
}

class DateTime(
    val date: Date,
    val hour: Int,
    val minute: Int,
    val second: Int
): Date(date.year, date.month, date.day) {
    
    // Don't do this, symmetric but not transitive
    override fun equals(other: Any?): Boolean =
        when (other) {
            is DateTime -> (date == other.date) && (hour == other.hour) && (minute == other.minute) && (second == other.second)
            is Date -> date == other
            else -> false
        }

    // ...
}
```

위 구현 방식의 문제점은, 'DateTime' 끼리 비교할 때와 'Date'와 'DateTime'를 비교할 때 고려하는 프로퍼티의 차이에 있다.  
같은 날짜를 가지고 있어도 서로 다른 시간 값을 가진 두 'DateTime' 객체는 서로 같지 않다고 판단하지만, 
이 두 객체는 같은 'Date' 객체와 비교 했을 때는 동일하다고 여겨진다. 이로 인해, 이들의 관계는 'Transitive'를 만족하지 않게 된다.

```kotlin
val o1 = DateTime(Date(1992, 10, 20), 12, 30, 0)
val o2 = Date(1992, 10, 20)
val o3 = DateTime(Date(1992, 10, 20), 14, 45, 30)

o1 == o2 // true
o2 == o3 // true
o1 == o3 // false <- So equality is not transitive
```

여기서 같은 타입의 객체끼리만 비교하는 제한 규칙이 효과를 발휘하지 못한 이유는 상속을 사용했기 때문이다.  
상속을 사용하는 것은 '리스코프 치환 원칙(LSP)'을 위반하는 행위로, 이 원칙은 하위 클래스가 상위 클래스의 자리를 대체할 수 있어야 한다는 것을 의미한다.  
따라서, 상속 대신 'Composition'을 사용하는 것이 좋다.

'Composition'을 실행할 때, 서로 다른 타입의 객체들을 비교하지 않도록 주의해야 한다.  
이러한 클래스들은 데이터를 저장하고 이렇게 표현하는 데 있어서 이상적인 사례로, 이 방법을 선택하는 것이 좋다.

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

o1.equals(o2)       // false
o2.equals(o3)       // false
o1 == o3            // false

o1.date.equals(o2)  // true
o2.equals(o3.date)  // true
o1.date == o3.date  // true
```

---

객체 간의 'equality'는 '일관적(Consistent)'이어야 한다.   
즉, 두 객체에 적용되는 'equality' 메서드는 객체 중 하나가 수정되지 않는 이상 항상 같은 결과를 내야 한다.  
불변 객체에 대해서는 이 결과가 변함없이 일정해야 한다.  

간단히 말해서, 'equals'가 객체의 상태를 변경하지 않는 순수함수로 동작하여, 그 결과가 오직 입력 값과 그 객체의 상태에만 의존해야 한다.  
앞서 'Time' 클래스에서 이 원칙을 어긴 사례를 살펴보았고, 이 규칙은 'java.net.URL.equals()'와 같이 이 원칙을 위반한 많은 사례를 찾아볼 수 있다.

---

'null'이 아닌 값 x에 대해, 'x.equals(null)'은 항상 'false'여야 한다.  
이는 'null' 값이 고유해야 하며, 어떤 객체도 'null'과 같다고 판단되어서는 안되기에 중요한 원칙이다.

---

## Problem with equals in URL

Java의 'java.net.URL' 클래스에서 'equals'의 설계는 큰 결함을 가지고 있다.  
두 'java.net.URL' 객체가 동일한지를 결정할 떄, 실제 네트워크 작업에 의존하게 된다.

즉, 두 호스트 이름이 같은 IP 주소로 변환될 수 있다면, 두 객체는 동일하다고 판단된다.  
이러한 설계는 네트워크 의존성을 도입함으로써, 객체의 동등성 판단을 불안정하고 예측 불가능하게 만든다.  
객체 간의 'equality'는 네트워크 상태와 무관하게 결정되어야 한다.

```kotlin
import java.net.URL

fun main() {
    val enWiki = URL("https://en.wikipedia.org/")
    val wiki = URL("https://wikipedia.org/")
  
    println(enWiki == wiki)   // true when internet turned on, false otherwise
}
```

위 해결책은 몇 가지 중요한 문제점들이 있다.

'Consistent' 부족하다. 예를 들어, 두 URL 객체의 'equality'는 네트워크 상태에 따라 달라질 수 있다.  
이는 네트워크 접근 가능 여부나, 호스트 이름의 IP 주소 변화에 따라 두 URL의 'equality'가 달라질 수 있다는 것을 의미한다.  
이는 'equality' 판단의 신뢰성을 떨어뜨린다.

네트워크가 느릴 수 있으며, 'equals'와 'hashCode'는 빠른 실행 속도를 기대한다.  
하지만 위 방식은 네트워크 호출을 필요로 하며, 특히 대규모 URL 리스트를 처리할 떄 성능 저하를 초래할 수 있다.  
또한, Android 같은 플랫폼에서는 메인 스레드에서의 네트워크 작업이 금지되어 있어, 추가적인 복잡성을 가져온다.

HTTP에서는 가상 호스팅을 통해 서로 다른 컨텐츠를 제공하는 사이트들이 동일한 IP 주소를 공유할 수 있다.  
따라서, IP 주소가 같다고해서 반드시 같은 컨텐츠를 제공하는 것이 아니다.  
이 설계는 가상 호스팅 환경을 고려하지 않아, 실제로는 관련 없는 두 URL을 동등하다고 잘못 판단할 수 있다.

## Implementing equals

'equals' 메서드를 직접 구현하는 것은 특별한 이유가 없다면 권장되지 않는다.  
기본적으로 제공되는 'equality' 또는 'data class'를 활용하는 것이 좋다.

커스텀 'equality'가 필요한 경우, 그 구현이 'Reflexive', 'Symmetric', 'Transitive', 'Consistent'을 갖추고 있는지 반드시 확인해야 한다.  
또한, 해당 클래스를 'final'로 선언하거나, 상속받는 클래스에서 'equality'의 방식을 변경하지 않도록 주의해야 한다.  
커스텀 'equality'를 구현하면서 상속을 지원하는 것은 매우 어려우며, 일부는 실질적으로 불가능하다고 주장한다.  
'data class'는 기본적으로 'final' 상태이다.