'Any' 클래스에서 'override' 할 수 있는 또 다른 메서드는 'hashCode'이다.  
'hashCode'는 'HashTable'이라는 데이터 구조에서 사용되며, 이는 다양한 컬렉션이나 알고리즘의 내부에서 사용된다.

## Hash table

'HashTable'이 등장한 배경을 살펴보자.   

개발자는 요소를 신속하게 추가하고 찾을 수 있는 컬렉션이 필요하다고 생각했고, 
이런 기능을 하는 컬렉션으로는 중복된 요소를 허용하지 않는 'set' 또는 'map'이 있다.  
따라서, 요소를 추가할 떄는 먼저 해당 요소와 동일한 요소가 있는지를 확인해야 한다.

배열 또는 연결 리스트 기반의 컬렉션은 포함된 요소를 확인하는 과정이 모든 요소를 차례대로 비교해야 하기 때문에 빠르지 않다.  
예를 들어, 수백만 개의 텍스트 조각이 담긴 배열이 있고, 이 중 특정 텍스트가 포함되어 있는지 확인해야 한다고 생각해보면, 이를 일일이 비교하는 것은 엄청난 시간이 필요할 것이다. 

바로 이 문제를 해결하는데 널리 사용되는 방법 중 하나가 'HashTable'이다. 필요한 것은 각 요소마다 번호를 부여하는 함수가 필요하다.   
이런 함수를 '해시 함수'라고 부르며, 같은 요소에 대해선 항상 동일한 값을 반환해야 한다.  
또한, '해시 함수'가 아래 조건을 만족하면 더욱 좋다.

- 처리 속도가 빠르다.
- 이상적으로, 서로 다른 요소에 대해 다른 값을 반환하거나, 최소한 충돌을 최소화하기 위한 충분한 변화를 가지고 있다.

해시 함수는 각 요소마다 번호를 할당하여 요소들을 다양한 '버킷'으로 분류한다.  
해시 함수에 설정한 요구 사항에 따라, 서로 같은 요소들은 항상 동일한 버킷에 배치될 것이다.

이 버킷들은 'HashTable'이라고 불리는 구조에 저장되는데, 'HashTable'은 버킷의 수와 동일한 크기의 배열이다.  
요소를 추가할 때마다, 해시 함수를 사용해 해당 요소가 어디에 배치되어야 할지를 계산하고 그 위치에 추가한다.  
이 과정은 해시를 계산하는 것이 빠르기 때문에 매우 신속하다. 

이 후 해시 함수의 결과를 배열의 인덱스로 사용하여 해당 버킷을 찾는다.   
요소를 찾을 떄는 같은 방식으로 해당 버킷을 찾고, 그 버킷 안의 요소 중 하나와 일치하는지만 확인한다.  
해시 함수는 동일한 요소에 대해 항상 같은 값을 반환해야 하므로 다른 버킷을 확인할 필요가 없다.  
이 방법으로, 적은 비용으로 요소를 찾는데 필요한 연산 횟수를 버킷의 수로 나눌 수 있다.  
예를 들어, 1,000,000개의 요소와 1,000개의 버킷이 있다면, 중복 검색 시 평균적으로 약 1,000개의 요소만 비교하면 되므로, 성능이 크게 향상된다.


좀 더 구체적인 예시로, 문자열과 4개의 버킷으로 분할되는 해시 함수가 있다고 가정해보자.

| Text                                           | Hash code |
|------------------------------------------------|-----------|
| "How much wood would a woodchuck chuck"        | 3         |
| "Peter Piper picked a peck of pickled peppers" | 2         |
| "Betty bought a bit of butter"                 | 1         |
| "She sells seashells by the seashore"          | 2         |

'HashCode'를 기반으로 다음과 같은 'HashTable'이 구성된다.

| Index | Object to which hash table points                                                       | 
|-------|-----------------------------------------------------------------------------------------|
| 0     | []                                                                                      |
| 1     | ["Betty bought a bit of butter"]                                                        |
| 2     | ["Peter Piper picked a peck of pickled peppers", "She sells seashells by the seashore"] |
| 3     | ["How much wood would a woodchuck chuck"]                                               |

이제, 'HashTable'에서 새로운 텍스트의 존재 여부를 확인할 때, 'HashCode'를 계산하게 된다.  

- 'HashCode'가 0인 경우, 이 텍스트는 목록에 없다고 볼 수 있다.  
- 'HashCode'가 1이나 3인 경우, 단 하나의 텍스트와 비교해야 한다.  
- 'HashCode'가 2인 경우, 두개의 텍스트와 비교해야 한다.

이러한 'HashTable' 개념은 기술 분야에서 인기 있다.   
데이터베이스, 다양한 인터넷 프로포콜, 그리고 많은 프로그래밍 언어의 표준 라이브러리 컬렉션에서 사용된다.  
'Kotlin/JVM'에서는 기본적으로 제공되는 'Set(LinkedHashSet)'와 'Map(LinkedHashMap)'도 이 원리를 활용한다.  
'HashCode'를 생성하기 위해서는 'hashCode()' 함수를 사용한다.

---

## Problem with mutability

요소에 대한 해시는 그 요소가 추가될 때에만 계산된다. 요소가 변경되어도 그 위치는 바뀌지 않는다.  
이로 인해, 'LinkedHashSet'과 'LinkedHashMap'에서 객체가 추가된 후 변경되면, 해당 객체는 제대로 동작하지 않게 된다.

```kotlin
data class FullName(
    var name: String,
    var surname: String
)

val person = FullName("John", "Smith")
val muSet = mutableSetOf<FullName>()
muSet.add(person)
person.surname = "Doe"

print(person)               // FullName(name=John, surname=Doe)
print(person in muSet)      // false
print(muSet)                // [FullName(name=John, surname=Doe)]
```

이 문제는 이미 'Item 1'에서 지적되었다.  
해시를 기반으로 한 데이터 구조나 그 외의 가변 프로퍼티를 기반으로 요소를 조직하는 모든 데이터 구조에서는 가변 객체의 사용을 피해야 한다.  
'Set'에서 가변 요소를 사용하거나, 'Map'의 'Key'로 사용하는 것은 피해야 하며, 특히 이런 컬렉션 내에서 요소를 변경해서는 안된다.  
이는 일반적으로 불변 객체를 사용해야 하는 강력한 이유 중 하나이다.

---

## The contract of hashCode

위 파트에서 'hashCode'의 필요성을 이해하면, 이 메서드가 어떤 행동을 해야하는지 분명해진다.  
아래는 'hashCode' 메서드에 대한 'contract'이다.

1. 동일한 객체에 대해 'hashCode'를 여러 번 호출하더라도, 'equals' 비교에 사용되는 정보가 변경되지 않는 한, 항상 같은 정수 값을 반환해야 한다.
2. 'equals'를 사용해 두 객체가 동등함을 확인했다면, 각각의 객체에 대해 'hashCode'를 호출했을 떄 동일한 정수 값을 반환해야 한다.

첫 번째 요구사항은 'hashCode'가 'consistent'해야 함을 의미한다.  

두 번째 요구사항은 개발자들이 간과하는 더욱 중요한 사항이다.  
'hashCode'는 항상 'equals'와 'consistent'해야 하며, 동등한 객체는 반드시 동일한 'HashCode' 값을 가져야 한다.  
만약 이 원칙이 지켜지지 않으면, 'HashTable'을 기반으로 하는 컬렉션에서 객체들이 분실될 위험이 있다.

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
print(FullName("John", "Smith") in muSet)                       // false
print(FullName("John", "Smith") == FullName("John", "Smith"))   // true
```

위와 같은 이유로 인해, Kotlin에서는 커스텀 'equals' 구현이 있는 경우, 'hashCode'도 함께 'overriding' 할 것을 권장하는 이유이다.

![img.png](hashcode_overriding.png)

또한, 요구되지는 않지만 이 기능을 실제로 효율적으로 만들기 위한 중요한 요구사항이 있다.  
이는 바로, 'hashCode'가 요소들을 가능한 넓은 범위에 걸쳐 분산시키는 것이다.  
이는 서로 다른 객체들이 서로 다른 해시 값에 할당될 가능성을 최대화함으로써, 해시 충돌을 최소화하고 데이터 구조의 성능을 극대화하는데 도움이 된다.

같은 버킷 안에 다양한 요소들이 과도하게 배치되는 상황을 생각해보자. 이 경우, 'HashTable'의 사용은 별다른 이점을 제공하지 않는다.   
예를 들어, 만약 'hashCode'가 항상 같은 값을 반환한다면, 이는 모든 요소를 동일한 버킷에 할당되게 만들 것이다.  
이는 기술적으로 'contract'를 만족할 지 몰라도, 실질적으로는 완전히 무용지물이다.

아래 예제를 보면, 제대로 구현된 'hashCode'와 항상 0을 반환하는 'hashCode'의 예시를 비교해보자.  
각각의 'equals' 사용 횟수를 계산하는 카운터를 추가하고, 두 타입의 값으로 구성된 'Set'에서 작업할 때 'Terrible' 클래스의 방식이 훨씬 더 많은 비교를 한다는 것을 알 수 있다. 


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

    override fun hashCode(): Int = 0        // Do not do that

    companion object {
        var equalsCounter = 0
    }
}

val properSet = List(10000) { Proper("$it") }.toSet()
print(Proper.equalsCounter)                 // 0

val terribleSet = List(10000) { Terrible("$it") }.toSet()
print(Terrible.equalsCounter)               // 50117047

Proper.equalsCounter = 0
println(Proper("9999") in properSet)        // true
println(Proper.equalsCounter)               // 1

Proper.equalsCounter = 0
println(Proper("A") in properSet)           // false
println(Proper.equalsCounter)               // 0

Terrible.equalsCounter = 0
println(Terrible("9999") in terribleSet)    // true
println(Terrible.equalsCounter)             // 2077

Terrible.equalsCounter = 0
println(Terrible("A") in terribleSet)       // false
println(Terrible.equalsCounter)             // 10001
```

---

## Implementing hashCode

Kotlin에서 '커스텀 equals'를 정의할 때에만 주로 'hashCode'를 정의한다.  
'data' modifier를 사용하면, 'equals'와 함께 일관성 있는 'hashCode'도 자동으로 생성된다.  
'커스텀 equals'가 없다면, 정확히 어떤 작업을 하고 있는지 알고 있으며 그것을 하기 위한 충분한 이유가 있는 경우가 아니면, '커스텀 hashCode'를 정의하지 않는 것이 좋다.  
'커스텀 equals'가 있다면, 동일한 요소에 대해서는 항상 같은 값을 반환하는 'hashCode'를 구현해야 한다.

객체의 중요한 프로퍼티들을 기반으로 'equality check'를 하는 'equals'를 구현한 경우, 이 프로퍼티들의 'HashCode'를 활용해 'hashCode'를 구현해야 한다.  
여러 'HashCode'를 하나의 'HashCode'로 합치는 일반적인 방법은 모든 'HashCode'를 하나의 결과 값에 누적시키는 것이다.  
이 때, 다음 'HashCode'를 추가할 때마다 현재 결과에 '31'을 곱한다. 반드시 '31'이어야 하는 것은 아니지만, 이 숫자는 해당 용도에 매우 적합한 특성을 가지고 있다.

'data class'에 의해 자동으로 생성된 'hashCode'도 이 관례를 따른다.  
아래는 'equals'와 함께 전형적인 'hashCode'의 구현 예시이다.

```kotlin
class DateTime(
    private var millis: Long = 0L,
    private var timezone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false

    override fun equals(other: Any?): Boolean =
        other is DateTime &&
                other.millis == millis &&
                other.timezone == timezone

    override fun hashCode(): Int {
        var result = millis.hashCode()
        result = result * 31 + timezone.hashCode()
        return result
    }
}
```

'Kotlin/JVM'에서 유용한 함수 중 하나는 해시를 계산하는 'Objects.hashCode'이다.

```kotlin
override fun hashCode(): Int = 
    Objects.hash(timeZone, millis)
```

Kotlin 표준 라이브러리에는 'Objects.hashCode'가 없지만, 다른 플랫폼에서 필요한 경우 직접 구현할 수 있다.

```kotlin
override fun hashCode(): Int =
    hashCodeOf(timezone, millis)

inline fun hashCodeOf(vararg values: Any?) = 
    values.fold(0) { acc, value ->
        (acc * 31) + value.hashCode()
    }
```

표준 라이브러리에 위와 같은 함수가 포함되지 않은 이유는, 실제로 'hashCode'를 직접 구현해야 할 상황이 드물기 때문이다.  
예를 들어, 앞서 언급된 'DateTime' 클래스의 경우, 'equals'와 'hashCode'를 직접 구현하기 보다는 'data' modifier를 사용하는 것이 더 간단할 것이다.

```kotlin
data class DateTime2(
    private var millis: Long = 0L,
    private var timezone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false
}
```

'hashCode'를 구현할 경우 반드시 기억해야 할 가장 중요한 원칙은, 'hashCode'가 'equals'와 항상 '일관(consistent)'되어야 하며,
동일하다고 판단되는 객체들에 대해서는 언제나 같은 'HashCode' 값을 반환해야 한다는 것이다.