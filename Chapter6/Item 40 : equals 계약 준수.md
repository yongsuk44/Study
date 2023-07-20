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
class Cat: Animal()

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