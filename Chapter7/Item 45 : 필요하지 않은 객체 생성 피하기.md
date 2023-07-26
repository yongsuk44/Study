# Item 45: 필요하지 않은 객체 생성 피하기

객체 생성은 항상 어떠한 비용이 발생하기에 불필요한 객체 생성을 피하는 것은 중요한 최적화 방법이 될 수 있습니다.

예시로 JVM에서 같은 문자열을 갖는 다른 객체가 동일한 VM에서 실행될 때, 문자열 객체를 재사용하는 것을 보장합니다.

```kotlin
val str1 = "Lorem ipsum dolor sit amet"
val str2 = "Lorem ipsum dolor sit amet"
println(str1 == str2) // true
println(str1 === str2) // true
```

작은 값을 지니는 기본형(`Integer`, `Long` 등)들도 JVM에서도 재사용을 보장합니다. (기본적으로 `Integer` 캐싱은 -128~127 숫자를 보유합니다.)

```kotlin
val i1: Int = 1
val i2: Int = 1
println(i1 == i2) // true
println(i1 === i2) // true, because i2 was taken from cache
```

위와 같이 재사용되는 객체들은 동일함을 보여줍니다. 그러나 -128보다 작거나 127보다 큰 숫자를 사용하면, 서로 다른 객체가 생성됩니다.

```kotlin
val i1: Int? = 128
val i2: Int? = 128
println(i1 == i2) // true
println(i1 === i2) // false
```

위 `Int`는 `null` 허용 타입으로 작성되어 내부적으로 `int` 대신 `Integer`가 사용되도록 강제 합니다.

컴파일 시 만약 `Int`를 사용하게되면 일반적으로 기본형 `int`로 사용되지만, `Int?`와 같이 `null`을 허용하거나 타입 인수(type argument)로 사용될 때는 `Integer`가 대신
사용됩니다.
이는 기본형(`Integer` or `Long` 등)이 `null`이 될 수 없고 타입 인수(type argument)로 사용될 수 없기 때문입니다.

---

## 객체 생성 비용

무언가를 객체로 래핑하는 경우 다음 3가지의 비용이 발생됩니다.

### 객체는 추가적인 공간을 차지함

모든 객체는 헤더 정보를 가지며 이 헤더는 객체의 유형, 상태, 동기화 정보, GC 상태 등을 저장합니다.

현대의 '64bit JDK'에서는 이 헤더가 12byte를 차지하고 8byte 배수로 패딩되므로 최소 객체 크기는 16byte가 됩니다.  
패딩은 메모리 내에서 객체가 효율적으로 배치되도록 하기 위한 것입니다. 추가로 '32bit JVM'에서는 헤더의 오버헤드가 8byte입니다.

객체는 객체 참조를 포함합니다. 이 참조는 해당 객체에 접근하는데 사용되는 주소 정보를 가집니다.  
일반적으로 참조는 '32bit 플랫폼' 혹은 '-Xmx32G까지의 64bit 플랫폼'에서는 4byte를 차지하며, 32Gb 이상의 메모리 할당(-Xmx32G 이상)에서는 8byte를 차지합니다.

이러한 수치들은 미미하게 보일 수 있지만, 객체가 많아질수록 누적되어 중요한 비용이 될 수 있습니다.
특히 정수와 같은 작은 요소들을 생각할 때, 이들은 유의미한 차이를 만들어줍니다.

기본형 `int`는 4byte를 차지하지만, 대부분 사용되는 '64bit JDK'에서는 래퍼 타입으로써 `Integer`는 16byte를 차히가게 됩니다.
이는 헤더(12byte) 이후의 4byte 공간에 들어가게됩니다. 추가적으로 `Integer`에 대한 참조 저장 시 추가로 8byte가 필요하게 됩니다.
이에 따라 기본형 `int` 대비 `Integer`는 거의 5배의 공간을 차지하게 됩니다.

이런식으로 불필요한 객체 생성은 메모리 사용량을 크게 늘릴 수 있으므로, 가능한 객체 생성을 피할 수 있으면 피하는 것이 효과적입니다.

### 요소 캡슐화 시 추가적인 함수 호출이 필요함

요소가 캡슐화되어 있을 때, 요소에 대한 접근 시 메서드를 호출하는 작은 비용이 발생하게 됩니다.
이는 단일 연산에서는 거의 무시해도 괜찮을 정도로 작지만, 대량의 데이터를 처리하거나 매우 빈번하게 접근해야 하는 경우 비용이 누적되어 성능에 영향을 줄 수 있습니다.

따라서 캡슐화는 OOP에서 중요한 원칙이지만 성능이 중요한 코드에서는 불필요한 객체 생성을 최소화하고, 필요한 경우 재사용을 통해 성능을 최적화하는 것이 중요합니다.

그러나 이는 'trade-off' 관계로 어느 정도의 성능향상을 위해 캡슐화를 너무 많이 제한하게 되면 코드의 유지보수성과 안정성이 떨어질 수 있음을 알고 있어야 합니다.

### 객체는 생성되어야 함

객체 생성은 프로그램에서 빈번하게 발생하지만, 일정한 비용을 수반하는것을 위 2가지를 통해 이해할 수 있습니다.
이 비용은 객체가 생성될 때마다 누적되므로, 많은 수의 객체를 다룰 때에는 성능에 영향을 줄 수 있으며 객체 생성과정은 다음과 같은 단계들이 포함됩니다.

#### 메모리 할당

객체를 위한 메모리 공간이 할당되어야 하며 이 공간은 객체의 데이터와 상태를 저장하는데 사용되며 'JVM heap' 영역에 위치합니다.
메모리 할당은 시스템 리소스를 사용하므로 비용이 발생합니다.

#### 생성자 호출

객체 생성 시 객체의 생성자가 호출됩니다. 생성자는 객체의 초기 사앹를 설정하는데 사용되며, 이 과정 역시 비용이 발생하게 됩니다.

#### 참조 생성

객체 생성 시 객체를 가리키는 참조가 생성됩니다. 이 참조는 메모리에 저장되며, 객체에 대한 후속 작업을 수행하는데 사용됩니다.

#### 초기화

객체의 필드와 속성은 해당 객체가 생성될 때 초기화됩니다.
이 초기화 과정은 생성자에서 수행되며, 초기화에 필요한 계산이나 메모리 할당 등으로 인해 비용이 추가로 발생될 수 있습니다.

```kotlin
class A

private val a = A()

// Benchmark result: 2.698 ns/op
fun accessA(blackHole: BlackHole) {
    blackHole.consume(a)
}

// Benchmark result: 3.814 ns/op
fun createA(blackHole: BlackHole) {
    blackHole.consume(a)
}

// Benchmark result: 3828.540 ns/op
fun createListAccessA(blackHole: BlackHole) {
    blackHole.consume(List(1000) { a })
}

// Benchmark result: 5322.857 ns/op
fun createListCreateA(blackHole: BlackHole) {
    blackHole.consume(List(1000) { A() })
}
```

객체 제거 시 위에서 언급한 객체 생성 비용 3가지를 모두 제거할 수 있습니다.
그리고 객체 재사용 시 첫 번째와 세번 쨰 비용을 제거할 수 있습니다.

이를 이해하고 있으면 코드에서 불필요한 객체 수를 제한하는 것에 대해 생각할 수 있으며 아래는 이를 어떤 방식으로 처리하는 방법 입니다.

---

## `object` 선언

객체를 매번 생성하는 대신 `object` 선언(싱글톤 패턴)을 사용 시 간단하게 객체를 재사용할 수 있습니다.  

아래 구현은 `LinkedList` 구현으로 비어있을 수 있고 요소를 포함할 수 있고 나머지를 가리키는 노드일 수 있습니다.

```kotlin
sealed class LinkedList<T>

class Node<T>(
    val head: T,
    val tail: LinkedList<T>
) : LinkedList<T>()

class Empty<T> : LinkedList<T>()

val list: LinkedList<Int> = Node(1, Node(2, Node(3, Empty())))
val list2: LinkedList<String> = Node("A", Node("B", Empty()))
```

이 구현은 리스트 생성 시 매번 `Empty` 인스턴스 생성하는것이며 이는 불필요한 객체 생성으로 이어질 수 있습니다.
이를 해결하기 위해 하나의 인스턴스만 생성하고 이를 재사용하도록 할 수 있습니다.

그러나 제네릭 타입으로 문제가 발생하는데 이와 같은 상황에서는 `Empty` 리스트가 모든 리스트의 하위 타입이 되도록 해야 합니다.
이를 위해 `Nothing`이라는 특수한 타입을 사용하면 됩니다.

이 `Nothing`은 모든 타입의 하위 타입으로 Kotlin에서 특별하게 제공되는 타입입니다.  
이로 인해, `LinkedList<Nothing>`은 모든 `LinkedList`의 하위 타입이되어 모든 타입의 리스트에 대해서 공통된 `Empty` 리스트를 표현할 수 있게 됩니다.

하지만 이러한 방식은 불변(immutable)객체에서만 사용해야 합니다. 만약 가변(mutable)객체에서 사용하게되면, 상태 공유로 인해 찾기 어려운 버그가 발생할 수 있습니다.

```kotlin
sealed class LinkedList<out T>

class Node<out T>(
    val head: T,
    val tail: LinkedList<T>
) : LinkedList<T>()

object Empty : LinkedList<Nothing>()

val list: LinkedList<Int> = Node(1, Node(2, Node(3, Empty)))
val list2: LinkedList<String> = Node("A", Node("B", Empty))
```

이처럼 `object` 선언을 통해 불필요한 객체 생성을 피할 수 있지만, 더 많은 방법들이 존재하며 팩토리 함수와 캐시를 사용하는 것입니다.

팩토리 함수는 객체 생성을 캡슐화하고 캐시는 이미 생성된 객체를 재사용하도록 합니다.