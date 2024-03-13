'compareTo'는 'Any' 클래스 안에 포함되어 있지 않다.  
Kotlin에서는 'compareTo'를 연산자로 취급하며, 이는 수학적 부등호와 같은 역할을 한다.

```kotlin
obj1 > obj2     // Translates to obj1.compareTo(obj2) > 0
obj1 < obj2     // Translates to obj1.compareTo(obj2) < 0
obj1 >= obj2    // Translates to obj1.compareTo(obj2) >= 0
obj1 <= obj2    // Translates to obj1.compareTo(obj2) <= 0
```

'compareTo'는 `Comparable<T>` 인터페이스에도 있다.  
어떤 객체가 `Comparable<T>` 인터페이스를 구현하거나, 'compareTo'라는 이름의 연산자 메서드를 가지고 있다면, 그 객체는 자연스러운 순서가 있다는 것을 의미한다.

이 순서는 다음과 같은 조건을 충족해야 한다.

- 'Antisymmetric'
  - 대칭적이지 않아야 한다.
  - 만약, 'a >= b'이고, 'b >= a' 라면 'a == b' 이다.
  - 따라서, 비교와 'equality' 사이에는 관계가 있으며, 이 둘은 서로 일관성(Consistent)을 가져야 한다.
- 'Transitive' 
  - 추이적이여야 한다.
  - 만약, 'a >= b'이고, 'b >= c' 라면 'a >= c' 이다.
  - 이와 마찬가지로, 'a > b'이고, 'b > c' 라면 'a > c' 이다.
  - 이러한 특성이 없으면, 정렬 알고리즘 중 일부에서는 정렬이 무한히 걸릴 수 있기 때문에 중요하다. 
- 'Connex' 
  - 연결적이여야 한다.
  - 'a >= b'이거나 'b >= a'와 같이 모든 두 요소 사이에 관계가 있어야 한다.
  - Kotlin에서는 'compareTo'가 'Int'를 반환하기 떄문에, 모든 비교 결과는 양수, 음수, 0 중 하나가 되는 것을 보장한다.
  - 이처럼 두 원소 사이에 관계가 없다면, 클래식 정렬 알고리즘(e.g : 퀵소트, 삽입 정렬 등)을 사용할 수 없게 되므로 중요하다.

---

## Do we need a compareTo?

Kotlin에서는 'compareTo'를 직접 구현할 필요가 거의 없다.  
한 가지 전역적인 자연스러운 정렬 순서를 설정하기 보단, 각각의 상황에 맞게 순서를 정하는 것이 더 많은 유연성을 제공한다.

예를 들어, 'sortedBy'를 사용하면 컬렉션을 정렬할 수 있으며, 비교가 가능한 'Key'를 제공하기만 하면 된다.  
아래 예제에서는 'surname'을 기준으로 정렬을 하고 있다.

```kotlin
class User(val name: String, val surName: String)
val names = listOf<User>(/* ... */)

val sorted = names.sortedBy { it.surName }
```

만약, 단순히 'Key'로 비교하는 것 보다 더 복잡한 비교가 필요하다면, 'sortedWith'을 사용할 수 있다.  
이 함수는 'comparator'를 통해 요소들을 정렬하는데, 'comparator'는 'compareBy' 함수를 사용하여 생성할 수 있다.  

예를 들어, 사용자들을 'surname'으로 먼저 정렬하고, 'surname'이 같으면 'name'으로 비교하여 정렬하는 방식이다.

```kotlin
val sorted = names.sortedWith(compareBy({ it.surName }, { it.name }))
```

물론, 'User' 클래스에 `Comparable<User>` 인터페이스를 구현할 수도 있지만, 어떤 기준으로 정렬해야 할지는 명확하지 않다.  
특정 프로퍼티에 따라 'User'를 정렬할 필요가 있을 때, 그 기준이 확실하지 않다면, 해당 객체들을 비교 가능한 상태로 만들어서는 안된다. 

'String'은 자연스러운 알파벳 순서를 가지고 있어, `Comparable<String>` 인터페이스를 구현한다.  
이는 텍스트를 알파벳 순으로 정렬하는 것과 같은 경우 매우 유용하다. 하지만, 이런 특성이 단점으로 작용할 때도 있다.  

예를 들어, 두 문자열을 부등호로 비교하는 것은 직관적이지 않게 느껴질 수 있다.  
대다수는 부등호를 사용한 문자열 비교를 보고 이해하기 어려울 것이다.

```kotlin
// Don't do this
println("Kotlin" > "Java")   // true
```

날짜와 시간과 같이 자연스러운 순서를 가진 객체들도 있다.  
하지만, 객체에 자연스러운 순서가 있는지 확실하지 않다면, 'comparator'를 사용하는 편이 더 좋다.

자주 사용하는 몇 가지 'comparator'가 있다면, 그것들을 클래스의 'companion object'에 배치할 수 있다.  
이렇게 하면 코드 관리가 편리해지고, 필요할 때 쉽게 접근할 수 있다.

```kotlin
class User(val name: String, val surName: String) {
    // ...
    
    companion object {
        val DISPLAY_ORDER = 
            compareBy(User::surname, User::name)
    }
}

val sorted = names.sortedWith(User.DISPLAY_ORDER)
```

---

## Implementing compareTo

'compareTo'를 직접 구현해야 할 때, 이 작업을 보다 쉽게 만들어 줄 수 있는 여러 Top-level 함수들이 존재한다.  
단지 두 값을 비교하는 것이 전부라면, 'compareValues'를 사용할 수 있다.

```kotlin
class User(
    val name: String,
    val surName: String
): Comparable<User> {
    
    override fun compareTo(other: User): Int =
        compareValues(surName, other.surName)
}
```

만약 더 많은 값을 사용해야 하거나, 특정한 기준에 따라 값을 비교해야 하는 경우에는 'compareValuesBy'를 사용하면 된다.

```kotlin
class User(
    val name: String,
    val surName: String
): Comparable<User> {
    
    override fun compareTo(other: User): Int =
        compareValuesBy(this, other, {it.surName}, {it.name})
}
```

위 함수는 필요로 하는 대다수의 'comparator'를 만들 수 있게 도와준다.  
만약 특별한 로직을 적용한 'comparator'를 구현해야 한다면, 다음 조건을 만족해야 한다.

- 비교하는 두 객체가 동일한 경우 0을 반환해야 한다.
- 비교 대상이 되는 객체(receiver)가 다른 객체(other)보다 크다면 양수를 반환해야 한다.
- 반대로, 비교 대상이 되는 객체(receiver)가 다른 객체(other)보다 작다면 음수를 반환해야 한다.

이러한 조건을 충족시킨 후에는, 'Antisymmetric', 'Transitive', 'Connex'을 만족하는지도 확인해야 한다.