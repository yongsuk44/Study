# Item 41: HashCode 계약 준수

`hashCode` 메서드는 `Any`에서 오버라이드 할 수 있는 메서드입니다.

`hashCode` 함수는 해시 테이블(`HashTable`)이라는 인기 있는 자료구조에서 사용됩니다.
이 해시 테이블은 다양한 컬렉션이나 알고리즘의 내부에서 사용됩니다.

## Hash table

개발자들은 때때로 빠르게 요소를 추가하고 검색할 수 있는 컬렉션이 필요합니다.

이렇게 만들어진 `set` 또는 `map` 같은 컬렉션은 중복을 허용하지 않기 때문에, 요소를 추가할 때마다 그 요소가 이미 존재하는지 확인해야 합니다.

배열(array)이나 연결된 요소(linked elements)를 기반으로 한 컬렉션은 모든 요소와 비교를 해야 하기에 이러한 확인 작업에 빠르지 않습니다.

이 문제를 해결하기 위한 방법 중 하나는 해시 테이블입니다.
해시 테이블 사용 시 해시 함수를 사용하며 해시 함수는 각 요소에 대해 숫자를 할당하는 역할을 합니다.

개발자들은 해시 함수에 다음과 같은 특징을 기대하고 사용합니다.

### 1. 빠른 동작

해시 함수는 빠르게 동작해야 합니다.   
해시 테이블과 같은 자료 구조는 해시 함수를 사용하여 데이터를 빠르게 추가하고 검색하기 때문에 빠른 해시 함수는 성능을 향상시키는 데 중요합니다.
또한 해시 함수가 빠르면 데이터 구조 전반의 성능에도 긍정적인 영향을 미칩니다.

### 2. 동등한 요소에 대해 동일한 해시 코드 반환

동일한 입력 데이터는 항상 동일한 해시 코드로 변환되어야 합니다.   
이는 해시 테이블의 동작에 중요한 영향을 미치는 요소입니다.
만약 같은 데이터에 대해 서로 다른 해시 코드가 반환되면 해당 데이터를 찾거나 제거하는 과정에서 문제가 발생할 수 있습니다.

### 3. 서로 다른 요소에 대해 다른 해시 코드 반환 또는 충돌 최소화

이상적으로, 서로 다른 입력 데이터는 서로 다른 해시 코드로 변환되어야 합니다.  
만약 서로 다른 데이터에 대해 같은 해시 코드가 반환되면 충돌이 발생하고, 이는 해시 테이블 성능을 저하시킬 수 있습니다.
따라서 해시 함수는 데이터가 고르게 분산되도록 설계되어야 합니다.

### 해시 함수

해시 함수는 각 요소를 다른 버킷(`bucket`)으로 분류하며, 이 때 각 요소에 번호를 할당합니다.

해시 함수가 올바르게 작동한다면, 동일한 요소는 항상 동일한 버킷에 들어갈 것입니다.
이러한 버킷은 해시 테이블 구조에서 관리되며 해시 테이블은 버킷 수와 동일한 크기의 배열이 됩니다.

요소 추가 시, 해시 함수를 사용해 그 요소가 어느 버킷에 들어가야 할지 계산하고 그 버킷에 요소를 추가합니다.
이는 매우 빠르게 처리되며, 해시 값을 계산한 후 그 값을 배열의 인덱스로 사용해 버킷을 찾을 수 있습니다.

요소 검색 시, 다른 버킷을 확인 할 필요 없이 해당 버킷을 찾고 그 안에서 해당 요소를 찾으면 됩니다.
왜냐하면 해시 함수는 동일한 요소에 대해 항상 동일한 값을 반환하기 때문입니다.

요소의 수를 버킷의 수로 나눠서 소수의 요소만을 확인해 요소를 찾을 수 있습니다.
예를 들어 1M 요소와 1K 버킷이 있다면 중복 검색을 위해 평균적으로 1K 요소를 비교해야 합니다.
이렇게 해서 성능을 크게 향상시킬 수 있습니다.

아래 표는 문자열에 대한 해시 함수가 4개의 버킷으로 문자열을 분류하는 경우를 보여줍니다.

| Text                                           | Hash code |
|------------------------------------------------|-----------|
| "How much wood would a woodchuck chuck"        | 3         |
| "Peter Piper picked a peck of pcikled peppers" | 2         |
| "Betty bought a bit of butter"                 | 1         |
| "She sells seashells by the seashore"          | 2         |

해시 코드에 따라 다음과 같은 해시 테이블로 구성됩니다.

| Index | Object to which hash table points                                                       | 
|-------|-----------------------------------------------------------------------------------------|
| 0     | []                                                                                      |
| 1     | ["Betty bought a bit of butter"]                                                        |
| 2     | ["Peter Piper picked a peck of pcikled peppers", "She sells seashells by the seashore"] |
| 3     | ["How much wood would a woodchuck chuck"]                                               |

새로운 텍스트가 이 해시 테이블에 있는지 확인하려면, 해당 텍스트의 해시 코드를 계산하며 다음과 같은 경우의 수를 가질 수 있습니다.

- 해시 코드가 0인 경우, 그 텍스트는 리스트에 없다는 것을 바로 알 수 있음
- 해시 코드가 1 또는 3인 경우, 그 텍스트를 한 개의 텍스트와 비교
- 해시 코드가 2인 경우, 두 개의 텍스트와 비교

이러한 해시 테이블은 데이터베이스, 여러 인터넷 프로토콜, 많은 언어의 표준 라이브러리 컬렉션에서 사용되고 있씁니다.
Kotlin/JVM에서는 기본적으로 제공되는 `set(LinkedHashSet)`과 `map(LinkedHashMap)`이 해시 테이블을 사용합니다.
해시 코드를 생성하기 위해서는 `hashCode()`를 사용하면 됩니다.

---

## 가변성에 의한 문제점

해시 테이블에 요소가 추가될 때에만 해시 값이 계산됩니다. 이 말은 요소가 변하더라도 그것이 이동하지 않다는 것을 의미하기도 합니다.
이러한 성질로 인해 `LinkedHashSet`이나 `LinkedHashMap`의 `Key`는 객체가 추가된 후 변하게 되면 제대로 작동되지 않는 문제가 발생할 수 있습니다.

```kotlin
data class FullName(
    var name: String,
    var surname: String
)

val person = FullName("John", "Smith")

val muSet = mutableSetOf<FullName>()
muSet.add(person)
person.surname = "Doe"

print(person) // FullName(name=John, surname=Doe)
print(person in muSet) // false
print(muSet) // [FullName(name=John, surname=Doe)]
```

위 코드에서 알 수 있듯이, `FullName` 객체의 `surname`이 변경되어도 그 객체는 집합에서 움직이지 않습니다.
결과적으로 해당 객체는 이제 집합에 존재하지 않는 것처럼 행동하게 됩니다.

가변성을 띄우는 객체는 해시 기반의 자료구조나 다른 어떤 데이터 구조에서도 사용되지 않아야 합니다.
`set`이나 `map`에 가변 요소를 넣는 것은 좋지 않으며, 특히 해당 컬렉션에 포함된 요소를 변경해서는 안됩니다.
이는 불변성을 띄우는 객체를 사용하는 이유 중 하나 입니다.

따라서 가변성이 필요한 경우에는 안전하게 처리하기 위한 추가적인 조치를 취해야 하며, 가능한 한 불변성을 유지하는 것이 좋습니다.

---

## hashCode 규약

`hashCode` 메서드는 객체의 해시 코드를 반환하는 함수로 이를 사용하기 위해서는 아래 규약을 지켜야 합니다.

### hashCode 일관성

동일 객체에 대한 `hashcode` 메서드가 여러 번 호출될 경우, 객체의 `equals` 메서드에 사용되는 정보가 변경되지 않는 이상 항상 같은 정수를 반환해야 합니다.

### `equals()`와 `hashCode()`의 일관성 유지

두 객체가 `equals` 메서드를 통해 같다고 판단되면, 두 객체의 `hashCode` 메서드는 항상 같은 결과를 반환해야 합니다.

만약 위 2가지 규약을 지키지 않으면, 해시 테이블 컬렉션에서 요소가 사라질 수 있습니다.

```kotlin
class FullName(
    var name: String,
    var surname: String
) {
    override fun equals(other: Any?): Boolean =
        other is FullName &&
                name == other.name &&
                surname == other.surname
}

val muSet = mutableSetOf<FullName>()
muSet.add(FullName("John", "Smith"))
print(FullName("John", "Smith") in muSet) // false
print(FullName("John", "Smith") == FullName("John", "Smith")) // true
```

Kotlin에서 `equals` 메서드 재정의하는 상황에서는 `hashCode` 메서드를 함께 오버라이드 하는 것이 좋습니다.

추가적으로 필수는 아니지만, `hashCode` 메서드를 잘 활용하려면 가능한 한 넓은 범위로 요소를 분산시켜야 합니다.
서로 다른 요소는 가능한 서로 다른 해시 값을 가져야 하는 것이 이상적입니다.

예를 들어, 항상 같은 숫자를 반환하는 `hashCode` 메서드는 모든 요소를 동일한 버킷에 넣게 됩니다.
이는 공식적인 규약을 충족하지만, 같은 버킷에 많은 수의 다른 요소가 모이는 상황은 해시 테이블의 장점이 사라지는 상황이 생기므로 필요가 없는 메서드로 볼 수 있습니다.

이를 통해, 적절하게 구현된 `hashCode` 함수와 충돌을 유발하는 함수의 중요성을 이해할 수 있습니다.
아래는 `hashCode`가 성능에 어떤 영향을 미치는지에 대한 예시 입니다.

```kotlin
class Proper(val name: String) {
    override fun equals(other: Any?): Boolean {
        equalsCounter++
        return other is Proper && name == other.name
    }

    override fun hashCode(): Int = name.hashCode()

    companion object {
        var equalsCounter = 0
    }
}

class Terrible(val name: String) {
    override fun equals(other: Any?): Boolean {
        equalsCounter++
        return other is Terrible && name == other.name
    }

    override fun hashCode(): Int = 0

    companion object {
        var equalsCounter = 0
    }
}

val properSet = List(10000) { Proper("$it") }.toSet()
print(Proper.equalsCounter) // 0

val terribleSet = List(10000) { Terrible("$it") }.toSet()
print(Terrible.equalsCounter) // 50117047

Proper.equalsCounter = 0
println(Proper("9999") in properSet) // true
println(Proper.equalsCounter) // 1

Proper.equalsCounter = 0
println(Proper("A") in properSet) // false
println(Proper.equalsCounter) // 0

Terrible.equalsCounter = 0
println(Terrible("9999") in terribleSet) // true
println(Terrible.equalsCounter) // 2077

Terrible.equalsCounter = 0
println(Terrible("A") in terribleSet) // false
println(Terrible.equalsCounter) // 10001
```

---

## hashCode 구현

`equals()`가 재정의되었다면, `hashCode()`도 재정의해야 합니다. 이를 통해 `equals`와 일관성 있는 해시 코드를 얻을 수 있습니다.
만약 `equals()`가 재정의되지 않았다면 명확한 이유가 있지 않으면 `hashCode()`도 재정의하지 않는 것이 좋습니다.

가령, 주요 속성들의 동일성을 확인하는 일반적인 `equals()`를 구현했다면, 이들 속성들의 해시 코드를 기반으로 하는 일반적인 `hashCode()`를 구현해야 합니다.

`hashCode()`로 생성된 해시 코드들을 하나로 합치는 방법은 모든 해시 코드를 결과에 누적하되, 다음 값을 추가할 때마다 결과를 `31`로 곱하는 것이 일반적인 방법입니다.
`31`이라는 수치는 고정된 규칙은 아니지만, 널리 사용되므로 이를 기본적인 규칙으로 생각할 수 있습니다.

또한 `data modifier`로 생성된 해시 코드도 이 규칙을 따르고 있습니다.

```kotlin
class DateTime(
    private var millis: Long = 0L,
    private var timezone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false

    override fun equals(other: Any?): Boolean =
        other is DateTime &&
                millis == other.millis &&
                timezone == other.timezone

    override fun hashCode(): Int =
        millis.hashCode() * 31 + timezone.hashCode() // == Objects.hashCode(millis, timezone)
}
```

Kotlin/JVM에서는 `Objects.hashCode` 함수를 사용해 다음과 같이 해시 코드를 계산할 수 있습니다.

Kotlin 표준 라이브러리에는 `Objects.hashCode` 함수가 없지만, 다른 플랫폼에서 필요한 경우에는 아래와 같이 직접 구현할 수 있습니다.

```kotlin
override fun hashCode(): Int = hashCodeFrom(timeZone, millis)

inline fun hashCodeOf(vararg values: Any?) = values.fold(0) { acc, value ->
    (acc * 31) + value.hashCode()
}
```

위와 같은 함수가 표준 라이브러리에 포함되지 않은 이유는 일반적으로 직접 `hashCode()`를 구현하는 것이 드물기 때문입니다.
예를 들어, 위의 `DateTime` 클래스의 경우, `equals`와 `hashCode`를 직접 구현하는 대신 `data` 변경자를 사용할 수 있습니다:

```kotlin
data class DateTime2(
    private var millis: Long = 0L,
    private var timezone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false
}
```

`hashCode()` 구현 시 중요한 점은, 해당 메서드가 항상 `equals` 메서드와 일관성을 유지하며 동일한 요소에 대해 동일한 값을 반환해야 한다는 것입니다.