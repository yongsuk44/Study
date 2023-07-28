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

---

## 팩토리 함수와 캐시

팩토리 함수 사용 시 생성자를 통해 객체를 생성하는 대신, 객체 생성을 제어하고 필요에 따라 캐시를 사용하여 이미 생성된 객체를 재사용할 수 있습니다. 
이를 통해 불필요한 객체 생성을 줄이고 시스템 리소스를 효율적으로 활용할 수 있습니다.

아래는 표준 라이브러리의 emptyList 구현입니다.

```kotlin
fun <T> List<T> emptyList() = EMPTY_LIST
```

위와 같이 간헐적으로 일련의 객체를 가지고 있으며, 그 중 하나를 반환하는 형태를 사용할 수 있습니다.

대표적으로 Coroutine에서 기본 디스패처인 `Dispatchers.Default`가 스레드 풀 역할을 합니다.
이는 많은 스레드를 생성하고 유지하고 있으며, 새로운 작업이 요청되면 사용 중이지 않은 스레드가 해당 작업을 처리하도록 선택됩니다.
이 방식을 사용하면 작업을 처리하기 위해 새로운 스레드를 생성하고 초기화하는 데 드는 비용을 절약할 수 있습니다.

데이터베이스 연결 풀도 비슷한 역할을 하고 있습니다. 데이터베이스 연결을 설정하는 것은 시간이 걸리고 리소스를 많이 사용하게 됩니다.
이를 위해 연결 풀은 미리 생성된 연결을 유지하고, 필요한 경우에 해당 연결을 재사용합니다.
이렇게 하면, 각 요청마다 새로운 연결을 설정할 필요가 없어 성능이 향상됩니다.

위 예시들과 같이 객체 생성에 비용이 많이 드는 경우와 동시에 몇개의 가변 객체를 사용해야 하는 경우, 객체 풀을 가지는 것도 좋은 해결책이 될 수 있습니다.

추가로 파라미터화된 팩토리 메소드에 대한 캐싱도 가능합니다. 이와 같은 경우에는 객체를 맵에 저장할 수 있습니다.

```kotlin
private val connections = mutableMapOf<String, Connection>()

fun getConnection(host: String) =
    connections.getOrPut(host) { createConnection(host) }
```

모든 순수 함수에 대해 캐싱 작업을 할 수 있으며 이를 메모리제이션(memorization)이라고 부릅니다.
아래는 피보나치 수열을 계산하는 함수입니다.

```kotlin
private val FIB_CACHE = mutableMapOf<Int, BigInteger>()

fun fib(n: Int): BigInteger = FIB_CACHE.getOrPut(n) {
    if (n <= 1) BigInteger.ONE else fib(n - 1) + fib(n - 2)
}
```

이 함수는 `n`번째 피보나치 수를 계산하는데, 이전에 계산된 수가 캐시에 저장되어 있다면 이를 사용하고, 그렇지 않은 경우 계산을 수행합니다.
이것은 클래식한 피보나치 구현 방식보다 훨씬 빠르게 동작하며 아래 표는 이를 비교한 결과입니다.

| type        | n = 100 | n = 200 | n = 300  | n = 400  
|-------------|---------|---------|----------|----------|
| fibIter     | 1997ns  | 5234 ns | 7008 ns  | 9727 ns  |
| fib (first) | 4413 ns | 9815 ns | 15484 ns | 22205 ns |
| fib (later) | 8 ns    | 8 ns    | 8 ns     | 8 ns     |

```kotlin
fun fibIter(n: Int): BigInteger {
    if (n<= 1) return BigInteger.ONE
    
    var p = BigInteger.ONE
    var pp = BigInteger.ONE
    
    for (i in 2..n) {
        val temp = p + pp
        pp = p
        p = temp
    }
    
    return p
}
```

캐싱에 대한 이러한 접근 방식에는 다음과 같은 장•단점이 존재합니다.

장점으로는 이전에 계산된 결과를 캐시에 저장하면 해당 결과를 다시 검색할 때 시간을 절약할 수 있습니다.
즉, 한번 계산된 결과는 거의 즉시 검색할 수 있으며 이는 계산 비용이 크고 반복적으로 수행되는 연산에 좋은 성능 향상을 이끌어낼 수 있습니다.

단점으로는 메모리 사용량이 증가하게 됩니다.
캐시에 값이 추가될 때마다 이 값은 메모리에 저장되어야 하며, 이는 시간이 지남에 따라 많은 메모리를 차지하게 됩니다.
이는 캐시에 저장된 값이 크거나 캐시에 많은 수의 항목이 있을 때 문제가 될 수 있습니다.

이러한 문제를 해결하기 위해 캐시에 저장된 값이 오래되면 제거하는 방식을 사용할 수 있습니다.
그러나 캐시에서 항목을 제거하는 것은 복잡할 수 있으며, 잘못 관리된 경우 다른 문제가 발생될 수 있습니다.

또 다른 방법으로는 `Soft Reference`를 사용하는 것 입니다. 
`Soft Reference`는 메모리가 부족할 때 GC에 의해 제거될 수 있는 참조 유형입니다. 이로 인해 메모리 부족 문제를 완화할 수 있습니다.

캐시 설계는 쉽지 않으며, 결국 성능과 메모리 사용량 사이의 균형을 찾아야하는 중요한 고려 사항입니다.
따라서, 캐시 사용 시 위 장•단점을 고려하여 사용해야합니다. 또한 성능 문제를 메모리 문제로 바꾸는 것은 좋지 않습니다.

---

## 큰 사이즈 객체를 외부로 이동하여 사용

리프팅(lifting) 기법은 반복 처리에서 무거운 연산을 최소화하는데 유용합니다.

아래 예제에서는 `Iterable<T>`의 각 요소가 최대값인지 판별하려는 경우를 보여줍니다. 
초기 함수에서는 `Iterable`의 각 요소마다 최대값을 찾게 되어 비효율적입니다.

```kotlin
fun <T: Comparable<T>> Iterable<T>.countMax(): Int = count { it == this.max() }
```

이를 최댓값을 1번만 계산하고 외부 범위에 저장하여 개선할 수 있습니다.
아래와 같이 변경하게 되면 loop 중 최댓값을 계산할 필요가 없어집니다.

```kotlin
fun <T: Comparable<T>> Iterable<T>.countMax(): Int {
    val max = this.max()
    return count { it == max }
}
```

### Regex, Lazy 초기화

아래 예제는 정규 표현식을 사용하여 문자열이 유효한 IP 주소인지 검증하는 함수를 보여줍니다. 

이 예제는 정규식 객체가 함수 호출이 될때마다 생성됩니다.
정규식 패턴의 컴파일은 복잡한 작업으로 성능에 영향을 줄 수 있기에 문제가 될 수 있습니다.

```kotlin
fun String.isValidIpAddress(): Boolean {
    return this.matches("\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex())
}
```

이를 정규식 객체를 최상위 범위에 생성하고 이를 재사용하는 방식으로 개선할 수 있습니다.

```kotlin
private val IS_VALID_EMAIL_REGEX = "\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex()

fun String.isValidIpAddress(): Boolean {
    return this.matches(IS_VALID_EMAIL_REGEX)
}
```

만약 위 함수가 다른 함수들과 같이 있고, 정규식 객체를 사용하지 않으면 생성되지 않도록 설정하려면 `lazy`를 통해 정규식을 초기화할 수 있습니다.

```kotlin
private val IS_VALID_EMAIL_REGEX by lazy {
    "\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex()
}
```

---

## Lazy 초기화

사이즈가 큰 객체 생성 시 이를 지연시키는 것이 좋습니다.

예를 들어 클래스 A가 B, C, D의 인스턴스를 필요로 하고 이들 모두 사이즈가 큰 객체라고 가정해보고 
이때 A를 생성할 때 B, C, D를 생성한다면, A 생성 시 B, C, D를 생성해야 하므로 A를 생성하는데 많은 비용이 발생하게 됩니다.

```kotlin
class A {
    val b = B()
    val c = D()
    val d = D()
}
```

이를 해결하기 위해 사이즈가 큰 객체 초기화를 지연(lazy) 할 수 있습니다.

```kotlin
class A {
    val b by lazy { B() }
    val c by lazy { D() }
    val d by lazy { D() }
}
```

이렇게 되면, 각 객체는 최초로 사용되는 시점 직전에 초기화를 진행합니다. 이는 객체 생성 비용이 누적되는 것이 아닌 분산되게 되므로 성능 향상을 이끌어 낼 수 있습니다.

그러나 위 `lazy` 방식을 통한 초기화는 메서드가 빠르게 작동해야하는 경우에 문제가 될 수 있습니다.
예를 들어 A 클래스가 HTTP 요청에 응답하는 백엔드의 컨트롤러로 가정해보면 빠르게 시작되긴 하겠지만, 첫 번째 요청에 모든 객체들이 초기화되어야 합니다.
이로 인해 첫 번째 요청의 응답 시간이 길어지게 되고 이는 앱 실행 시간과 관계없이 발생되게 됩니다.

---

## 원시 타입 사용 시 성능 향상

JVM에서 숫자나 문자와 같은 기본 요소를 나타내기 위한 내장형 타입이 있고 이를 원시(primitives) 타입이라고 부릅니다.

가능한 경우 Kotlin/JVM 컴파일러에 의해 내부적으로 사용됩니다. 그러나 아래와 같은 경우 래퍼 클래스를 대신 사용해야 하는 경우가 있습니다.

1. `Nullable` 타입에 대해 연산할 때
2. 제네릭 타입을 사용할 때

JVM에서는 원시 타입(`int`, `long`, `float`)을 최적화하여 처리합니다. 
래핑 타입은 원시 타입을 객체로 감싸서 사용하는 것을 말하며 `Integer`, `Long`, `Float` 등이 있습니다.

원시 타입은 메모리 사용이 감싸진 타입보다 훨씬 효율적이므로, 성능이 중요한 경우 원시 타입을 사용하면 코드 실행 속도를 향상시킬 수 있습니다.
반복적인 숫자 연산이 일어나는 곳이거나 큰 컬렉션을 다루는 곳 등이 해당될 수 있습니다.

그러나 이러한 최적화는 항상 좋은점만 있는것이 아닙니다. 래핑 타입을 원시 타입으로 바꾸면서 코드의 가독성이 떨어질 수 있고, 
`null` 값을 다루어야 하는 경우에는 원시 타입은 `null` 값을 가질 수 없기에 래핑 타입을 사용해야 합니다.

그러므로 성능이 중요한 부분인지 아닌지를 잘 판단하고 성능이 중요한 부분에서는 이러 최적화를 적용하는 것이 좋습니다.
프로파일러를 사용하면 코드의 어느 부분이 성능에 가장 영향을 미치는지 알 수 있으며, 이를 통해 어떤 부분을 최적화할지 결정할 수 있습니다.

아래 예시는 컬렉션에서 요소가 비어있으면 `null`을 아니면 최대 값을 반환하는 함수 입니다.

```kotlin
fun Iterable<Int>.maxOrNull(): Int? {
    var max: Int? = null
    
    for (i in this) {
        max = if(i > (max ?: Int.MIN_VALUE)) i else max 
    }
    
    return max 
}
```

위 구현에는 2가지 단점이 있습니다.
1. 각 단계에서 엘비스(?:) 연산자를 사용합니다.
2. `null` 가능 값을 사용하므로 JVM 내부에서는 `int` 대신 `Integer`를 사용합니다. 

위 2가지 단점을 해결하려면 `while`을 사용하여 반복을 구현하면 됩니다.

```kotlin
fun Iterable<Int>.maxOrNull(): Int? {
    val iterator = this.iterator()
    if (!iterator.hasNext()) return null
    
    var max = iterator.next()
    
    while (iterator.hasNext()) {
        val e = iterator.next()
        if (e > max) max = e
    }
    
    return max
}
```
```