객체를 생성하는 과정은 때때로 비용이 많이 들며, 항상 어느 정도의 비용이 수반된다.   
이러한 이유로, 불필요한 객체 생성을 피하는 것은 중요한 최적화 방법 중 하나이다.

이 최적화는 다양한 단계에서 이루어질 수 있다.  
특히, JVM 환경에서는 동일한 가상 머신에서 실행되는 어떠한 코드가 같은 문자열 리터럴을 사용하는 경우, 해당 문자열 객체가 재사용됨이 보장된다.

```kotlin
val str1 = "Lorem ipsum dolor sit amet"
val str2 = "Lorem ipsum dolor sit amet"
println(str1 == str2)   // true
println(str1 === str2)  // true
```

JVM 내에서, 박싱된 원시 타입인 'Integer', 'Long'과 같은 객체들은 그 값이 작을 때 재사용된다.  
기본적으로, 'Integer'의 경우 캐시를 활용하여 '-128 ~ 127' 사이의 범위 내에서 숫자들이 필요할 때마다 이미 생성된 인스턴스를 재사용한다.

```kotlin
val i1: Int = 1
val i2: Int = 1
println(i1 == i2)   // true
println(i1 === i2)  // true, because 'i2' was taken from cache
```

위와 같이 객체 'equality'를 확인함으로써 동일한 객체임을 알 수 있다.  
그러나, -128 보다 작거나 127보다 큰 숫자를 사용하는 경우, JVM은 서로 다른 객체를 생성하게 된다.

```kotlin
val i1: Int? = 128
val i2: Int? = 128
println(i1 == i2)   // true
println(i1 === i2)  // false
```

위와 같이 'nullable' 타입을 사용하면 Java 내부적으로 기본형 'int'가 아닌, 'Integer'를 사용하도록 강제할 수 있다.  
보통 'Int' 사용 시, 기본적으로 원시 타입 'int'로 컴파일 되지만, 이를 'nullable' 타입으로 지정하거나 타입 인자로 사용하는 경우엔 'Integer'를 사용하게 된다.
이는 원시 타입이 'null' 값을 가질 수 없고, 타입 인자로도 사용될 수 없기 때문이다.
---

## Is object creation expensive?

무언가를 객체로 래핑하는 경우 다음 세 가지 비용이 든다.

1. 모든 객체는 헤더 정보를 가지며, 이 헤더는 객체의 타입, 상태, 동기화 정보, GC 상태 등을 저장한다.  
   현대의 64비트 JDK에서의 객체는 이 헤더가 12바이트를 차지하고, 8바이트 배수로 패딩되므로 최소 객체의 크기는 16바이트가 된다.  
   패딩은 메모리 내에서 객체가 효율적으로 배치되도록 하기 위한 것이다.
   또한 객체는 객체 참조를 포함하며, 이 참조는 해당 객체에 접근하는데 사용되는 주소 정보를 가진다.
   일반적으로 참조는 32비트 플랫폼에서는 4바이트, 64비트 플랫폼에서는 '-Xmx32G'까지는 4바이트, 그 이상에서는 8바이트가 된다.
   이러한 수치들은 작아 보일 수 있지만, 객체가 많아질수록 누적되어 큰 비용으로 이어질 수 있다.    
   예를 들어, 정수와 같이 작은 요소들에 대해 생각해보면, 이 차이가 중요하다.
   원시 타입 'Int'는 4바이트를 차지하지만, 대부분 사용되는 64비트 JDK에서는 래핑된 타입인 'Integer'를 사용하여 16바이트의 공간을 차지하게 된다.
   이는 헤더(12바이트) 이후의 4바이트 공간에 들어가게 된다. 추가적으로 'Integer'에 대한 참조 저장 시 추가로 8바이트가 필요하게 된다.
   이에 따라 기본형 'Int' 대비 'Integer'는 거의 5배의 공간을 차지하게 된다.

2. 요소가 캡슐화될 때, 요소에 대한 접근은 추가적인 함수 호출을 필요로 한다. 이는 함수 호출이 상당히 빠르기 때문에 비교적 작은 비용으로 여겨지지만,
   대량의 객체를 다루어야 할 경우 이러한 작은 비용들이 모여 상당한 부담이 될 수 있다.
   따라서 캡슐화를 무조건 제한하는 것이 아니라, 코드의 성능에 중요한 영향을 미치는 부분에서는 불필요한 객체 생성을 피하는 것이 좋다.

3. 객체 생성은 필수적인 과정이다. 이 과정에는 객체를 메모리에 할당하고, 그에 대한 참조를 생성하는 등의 작업이 포함된다.  
   각각의 작업은 비용이 매우 적을 수 있으나, 이런 작은 비용들이 누적될 경우 전체적인 성능에 부담을 줄 수 있다.

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

객체를 제거하면 위 세 가지 비용을 모두 피할 수 있다. 객체를 재사용하면 첫 번째와 세 번째 비용을 제거할 수 있다.  
이 사실을 알면 코드에서 불필요한 객체의 수를 제한하는 것에 대해 생각해 볼 수 있다.

이를 위한 몇 가지 방법을 살펴보자.

---

## Object declaration

객체를 매번 새로 생성하는 대신에 재사용할 수 있는 간단한 방법 중 하나는 'object declaration('싱글톤)'을 사용하는 것이다.

예를 들어, 연결 리스트를 구현해야 한다고 가정해보자. 연결 리스트는 빈 상태이거나, 하나의 요소와 그 요소가 가리키는 나머지 리스트를 포함하는 노드로 구성될 수 있다.

이는 다음과 같이 구현될 수 있다.

```kotlin
sealed class LinkedList<T> {
    class Empty<T> : LinkedList<T>()
    class Node<T>(val head: T, val tail: LinkedList<T>) : LinkedList<T>()
}

// usage
val list: LinkedList<Int> = Node(1, Node(2, Node(3, Empty())))
val list2: LinkedList<String> = Node("A", Node("B", Empty()))
```

위 구현 방식에는 한 가지 문제가 있다. 바로, 리스트를 생성할 때마다 'Empty' 인스턴스를 새로 만들어야 한다는 점이다.  
이 문제를 해결하기 위해서는 하나의 인스턴스만을 생성하여 공통적으로 사용하는 것이 좋다.

여기서 유일한 문제는 제네릭 타입이다. 위 구현에서 'Empty 리스트'는 모든 리스트의 하위 타입으로 만들고 싶어한다.
하지만, 프로그래밍에서는 모든 가능한 타입의 리스트를 따로 지정할 수 도 없고, 실제로 그럴 필요도 없다.
이에 따른 해결책은 'Empty 리스트'를 'Nothing' 타입의 리스트로 만드는 것이다.
'Nothing'은 모든 타입의 하위 타입이기에, 리스트가 공변적(covariant)일 경우(즉, 'out' modifier를 가질 때), `LinkedList<Nothing>`은 모든 'LinkedList'의 하위
타입이 된다.
이처럼, 리스트가 불변이고 제네릭 타입이 오직 'out position'에서만 사용되기 때문에, 타입 인자를 공변적으로 만드는 것은 매우 적절한 선택이다.

```kotlin
sealed class LinkedList<out T> {
    data object Empty : LinkedList<Nothing>()
    class Node<out T>(val head: T, val tail: LinkedList<T>) : LinkedList<T>()
}

// usage
val list: LinkedList<Int> = Node(1, Node(2, Node(3, Empty)))
val list2: LinkedList<String> = Node("A", Node("B", Empty))
```

이 방법은 특히 불변성을 갖는 'sealed class'를 정의할 때 많이 자주 사용되는 유용한 기술이다.  
불변 객체를 사용하는 것이 중요한 이유는, 가변 객체를 사용할 경우 공유 상태를 관리하다가 탐지하기 어려운 버그로 이어질 수 있기 때문이다.

일반적으로, 가변 객체는 캐싱되어서는 안된다는 것이 기본 원칙이다(Item 1에서 설명했듯이).  
객체 선언 말고도, 객체를 재사용하기 위한 여러 방법이 있으며, 그 중 하나가 캐시 기능을 포함한 팩토리 함수를 사용하는 것이다.

---

## Factory function with a cache

'constructor'를 사용할 때마다 새로운 객체가 만들어진다. 그러나 팩토리 함수를 사용하는 경우에는 이와 다를 수 있다.  
팩토리 함수에는 캐시 기능을 추가할 수 있으며, 가장 간단한 사례로는 팩토리 함수가 항상 동일한 객체를 반환하는 것이다.

예를 들어, 아래는 표준 라이브러리의 'emptyList'가 구현된 방식이다.

```kotlin
fun <T> emptyList(): List<T> = EmptyList
```

때때로 여러 객체 중에서 하나를 선택해 반환해야 할 때가 있다.

예를 들어, Kotlin Coroutine 라이브러리에서 제공하는 기본 디스패처인 'Dispatchers.Default'는 여러 스레드를 관리하는 풀을 사용한다.  
이를 통해, 새로운 작업이 요청되면 사용 중이지 않은 스레드가 해당 작업을 처리하도록 선택된다.
이런 방식은 작업을 처리하기 위해 새로운 스레드를 생성하고 초기화하는 비용을 절약할 수 있다.

이와 유사하게, 데이터베이스의 연결을 설정하는 것은 시간이 걸리고 리소스를 많이 사용한다.  
이를 위해, 데이터베이스 연결 풀을 운영하여 미리 생성된 연결을 유지하고, 필요한 경우에 해당 연결을 재사용할 수 있다.
이렇게 하면, 각 요청마다 새로운 연결을 설정할 필요가 없어 성능이 향상됩니다.

또한, 객체 생성 비용이 크거나 동시에 여러 개의 가변 객체를 사용해야 할 수도 있는 경우, 객체 풀을 사용하는 것이 좋은 해결책이 될 수 있다.

파라미터화된 팩토리 함수에 대해서도 캐싱을 수행할 수 있다.    
이러한 경우 객체를 'Map'에 보관할 수 있다.

```kotlin
private val connections = mutableMapOf<String, Connection>()

fun getConnection(host: String) =
    connections.getOrPut(host) { createConnection(host) }
```

캐싱은 모든 순수 함수에 사용될 수 있으며, 이러한 경우를 'memorization'이라고 부른다.  
예를 들어, 주어진 위치의 피보나치 수를 정의에 따라 계산하는 함수가 있다.

```kotlin
private val FIB_CACHE = mutableMapOf<Int, BigInteger>()

fun fib(n: Int): BigInteger = FIB_CACHE.getOrPut(n) {
    if (n <= 1) BigInteger.ONE else fib(n - 1) + fib(n - 2)
}
```

위 메서드는 처음 실행될 때는 선형 솔루션만큼이나 효율적이며, 이미 계산이 된 경우에는 즉각적으로 결과를 반환한다.  
이 방식과 기존의 선형 피보나치 방식을 비교한 결과는 아래의 그래프와 표에 자세히 설명되어 있다.

또한, 이와 비교한 반복적 구현 방법도 함께 제시되어 있다.

|    type     | n = 100 | n = 200 | n = 300  | n = 400  |
|:-----------:|:-------:|:-------:|:--------:|:--------:|
|   fibIter   | 1997ns  | 5234 ns | 7008 ns  | 9727 ns  |
| fib (first) | 4413 ns | 9815 ns | 15484 ns | 22205 ns |
| fib (later) |  8 ns   |  8 ns   |   8 ns   |   8 ns   |

```kotlin
fun fibIter(n: Int): BigInteger {
    if (n <= 1) return BigInteger.ONE

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

'fib' 함수를 처음 사용할 때는 캐시 내에 값이 있는지 확인하고 그 곳에 값을 추가하는 등의 추가 작업 때문에 기존의 방식 보다 다소 느리다.  
하지만, 일단 값들이 캐시에 추가되고 나면, 데이터 검색 시간은 거의 즉각적으로 단축된다.

이 방식에는 한 가지 큰 단점이 있다. 바로, 데이터를 저장하기 위해 'Map'을 사용함으로써 더 많은 메모리를 차지하게 된다.  
만약, 이 데이터가 어느 시점에서 정리된다면 문제가 되지 않지만, 'GC'는 캐시 데이터와 미래에 필요 할 수 있는 다른 'static field'를 구분하지 않는다.
따라서, 'fib' 함수를 다시 사용하지 않는다고 해도, 가능한 한 오래 데이터를 유지할 것이다.
이 문제에 대한 한 가지 해결책은 메모리가 필요할 때 GC가 제거할 수 있는 'soft-reference'를 사용하는 것이다.
'soft-reference'와 'weak-reference'와는 다른 개념이며, 간단하게 이 둘의 차이점은 다음과 같다.

- 'weak reference'를 사용하면 GC가 해당 값을 언제든지 수집할 수 있도록 하여, 더 이상 사용되지 않는 개체가 메모리를 불피룡하게 차지하는 것을 방지한다.
- 'soft reference'는 메모리 요구가 있을 때까지 GC에 의해 객체가 수집되지 않도록 함으로써, 필요할 때까지 데이터를 유지할 수 있도록 한다.
이는 캐시 구현 시 매우 이상적인 선택이다.

캐시를 잘 설계하는 것은 쉽지 않으며, 결국 캐싱은 항상 성능과 메모리 사용 사이의 밸런스를 맞추는 'trade-off'가 된다. 
이 점을 기억하고, 캐시를 현명하게 활용해야 한다. 성능 개선을 위해 캐시를 도입했다가 메모리 부족으로 인한 새로운 문제를 직면하고 싶은 사람을 없을 것이다.

---

## Heavy object lifting

성능 개선을 위한 유용한 방법 중 하나는 무거운 객체를 외부 스코프로 옮기는 것이다.  
가능하다면, 컬렉션 처리 함수에서 수행되는 무거운 연산들을 일반적인 처리 영역으로 옮기는 것이 좋다.

예를 들어, 아래 함수에서는 'Iterable' 내의 각 요소마다 최대값을 찾아야 한다는 것을 알 수 있다.

```kotlin
fun <T : Comparable<T>> Iterable<T>.countMax(): Int = count { it == this.max() }
```

이를 개선하는 방법은 최대값을 찾는 연산을 'countMax' 함수 수준으로 이동시킴으로써, 연산이 한 번만 수행되도록 한다.  
이렇게 하면, 최대값을 찾는 비용이 한 번만 발생하고, 이후의 각 요소 비교는 이미 찾아진 최대값과만 이루어져 성능이 향상된다.

```kotlin
fun <T : Comparable<T>> Iterable<T>.countMax(): Int {
    val max = this.max()
    return count { it == max }
}
```

이처럼, 값을 계산하는 작업을 외부 스코프로 추출하여 불필요한 계산을 피하는것은 중요한 전략이다.  
이 방법이 당연해 보일 수 있지만, 실제로는 항상 그렇게 명확하지 않다.

예를 들어, 아래 함수를 보면 정규 표현식을 사용하여 문자열이 유효한 IP 주소를 포함하고 있는지 검증한다.

```kotlin
fun String.isValidIpAddress(): Boolean =
    matches("\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex())

// usage
print("5.173.80.254".isValidIpAddress())  // true
```

이 함수의 문제점은 정규 표현식 객체를 매번 사용할 때마다 새로 생성해야 한다는 점이다.  
정규식 패턴을 컴파일하는 것은 복잡한 연산 과정을 거치기에, 이는 성능에 영향을 줄 수 있다.  
이러한 이유로, 이 함수는 성능에 민감한 코드 영역에서 반복적으로 사용하기 적합하지 않다.  
하지만, 정규 표현식을 코드의 Top-level로 올림으로써 이를 개선할 수 있다.

```kotlin
private val IS_VALID_EMAIL_REGEX =
    "\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex()

fun String.isValidIpAddress(): Boolean = matches(IS_VALID_EMAIL_REGEX)
```

만약 이 함수가 다른 함수들과 같은 파일내에 위치하고 있고, 이 정규 표현식 객체가 실제로 사용되지 않는다면, 객체를 생성하고 싶지 않을 수 있다.  
이런 상황에서는 정규 표현식을 'lazy'하게 초기화하는 것이 좋다. 

```kotlin
private val IS_VALID_EMAIL_REGEX by lazy {
    "\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex()
}
```

---

## Lazy initialization

때때로 비교적 크고 복잡한 클래스를 생성해야 할 필요가 있을 때, 이를 'lazy'하게 수행하는 것이 좋다.  
예를 들어, 'A' 클래스가 'B','C','D'와 같이 크고 복잡한 인스턴스들을 필요로 한다고 가정해보자.  
이 인스턴스들을 'A'가 생성되는 시점에 모두 생성한다면, 'A' 생성 과정은 상당히 무섭고 오래 걸릴 것이다.  
이는 'B','C','D'를 생성하는 과정과 'A'의 나머지 부분을 구성하는 작업이 누적되기 때문이다.

```kotlin
class A {
    val b = B()
    val c = D()
    val d = D()
}
```

이를 해결하기 위한 방법은 'lazy'를 사용하여 큰 객체들을 게으르게 초기화하는 것이다.

```kotlin
class A {
    val b by lazy { B() }
    val c by lazy { D() }
    val d by lazy { D() }
}
```

이렇게 되면, 각 객체는 첫 사용 시점 바로 이전에 초기화된다.  
이로 인해, 객체 생성에 따른 비용이 한 번에 몰리는 것이 아닌, 시간에 걸쳐 분산되어 부담을 줄일 수 있다.

이 방식이 갖는 이점만큼 위험성도 있음을 잊으면 안된다.  
객체 생성 과정이 복잡하고 시간이 많이 걸릴 수 있음에도 불구하고, 때로는 객체가 빠르게 생성되어야 할 필요가 있다.  
예를 들어, HTTP 요청에 대응하는 백엔드 애플리케이션의 'Controller A'를 생각해보자.  
이 경우, 애플리케이션은 신속하게 시작되지만, 첫 번째 요청이 처리될 때 모든 'heavy' 객체들이 초기화되어야 한다.  
결과적으로, 첫 번째 요청에 대한 응답을 받기까지 상당한 시간이 지연되며, 애플리케이션이 얼마나 장시간 동안 실행되든 그 시간은 중요하지 않게 된다.  
이러한 상황은 바람직하지 않으며, 성능 테스트 결과에도 부정적인 영향을 미칠 수 있다.

---

## Using primitives

JVM에는 숫자나 문자 같은 기본적인 요소를 표현하기 위한 특별한 내장 타입이 존재한다.  
이를 'primitives'라고 부르며, Kotlin/JVM 컴파일러는 가능한 모든 경우에 내부적으로 이를 활용한다.  
그럼에도 불구하고, 일부 상황에서는 래핑된 클래스를 사용해야만 하는 경우도 있다.

1. 'nullable' 타입에 연산을 수행할 경우 ('primitives' 타입은 'null' 값을 가질 수 없음)
2. 타입을 제네릭으로 사용하는 경우

이를 알고 있으면, 내부적으로 'primitives'를 활용하여 래핑된 타입보다 코드를 최적화 할 수 있다.  
이런 최적화는 주로 Kotlin/JVM과 Kotlin/Native의 특정 변형에서만 유의미하며, Kotlin/JS에서는 전혀 의미가 없다.
또한, 숫자 연산이 반복적으로 수행될 때만 이 최적화가 유의미하다는 점을 명심해야 한다.
'primitives'와 래핑된 타입 모두 접근 속도는 다른 연산들에 비해 상당히 빠르지만, 차이는 주로 큰 컬렉션을 처리할 때나 객체를 집중적으로 다룰 때 드러난다.
강제적인 변경은 코드의 가독성을 해칠 수 있으니, 이러한 최적화는 성능이 중요한 코드 부분과 라이브러리에 한정해서 고려해야 한다.
어떤 부분이 성능에 중요한지는 'profiler'를 통해 파악할 수 있다.

아래 예제를 통해 Kotlin 표준 라이브러리를 구현한다고 가정해보자.  
이러한 상황에서, 'Iterable' 컬렉션이 비어 있으면 'null'을 반환하고, 그렇지 않으면 최대 요소를 반환하는 함수가 있다.  
'Iterable' 컬렉션을 한 번만 순회하면서 이 문제를 해결하는 것은 간단하지 않다. 하지만 아래와 같이 함수를 사용하면 이 문제를 해결할 수 있다.

```kotlin
fun Iterable<Int>.maxOrNull(): Int? {
    var max: Int? = null
    for (i in this) {
        max = if (i > (max ?: Int.MIN_VALUE)) i else max
    }
    return max
}
```

위 구현에는 몇 가지 중요한 단점이 있다.

1. 모든 단계마다 'Elvis(?:)' 연산자를 사용해야 하는 번거로움이 있다.
2. 'nullable' 값을 사용함으로써, JVM 내부적으로는 'int'가 아닌 'Integer'를 사용하게 된다.

이러한 문제들을 해결하려면, 'while' 루프를 사용한 반복 구현이 필요하다.

```kotlin
fun Iterable<Int>.maxOrNull(): Int? {
    val iterator = this.iterator()
    if (!iterator.hasNext()) return null

    var max: Int = iterator.next()

    while (iterator.hasNext()) {
        val e = iterator.next()
        if (e > max) max = e
    }

    return max
}
```

1 ~ 1,000만까지의 요소를 포함하는 컬렉션을 처리할 때, 최적화된 구현 방식은 약 300ms가 걸렸고, 이전 방식은 550ms가 걸렸다.  
이는 거의 2배의 차이로, 훨씬 더 빠른것을 볼 수 있다. 하지만, 이러한 차이를 보여주기 위한 극단적인 사례임을 기억해야 한다.
성능이 중요하지 않은 코드에서는 이러한 최적화는 합리적이지 않다. 
그렇지만, Kotlin 표준 라이브러리를 구현할 때는 성능이 중요하므로, 두 번째 접근 방식이 선택되었다.

```kotlin
public fun <T: Comparable<T>> Iterable<T>.max(): T? {
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