Kotlin은 프로그램을 모듈로 설계하고 각 모듈은 'class', 'object', 'type alias', 'functions', 'Top-level properties'와 같은 다양한 '요소'로 구성되며,
이 중 일부는 'read-write' 타입 `var` 또는 'mutable object'를 사용함으로써 '**상태 저장이 가능**'하다.

(e.g : `var a = 10`, `val list: MutableList<Int> = mutableListOf()` )

상태를 보유한 요소는 시간에 따라 변할 수 있는 속성을 가지고 있기에 현 상태에 이르게된 히스토리(이벤트, 변경 등)에 따라 요소의 동작이 달라질 수 있다.
아래는 상태를 보유한 클래스의 전형적인 예시로, 일정한 금액의 잔액을 갖는 `BankAccount` 클래스임.

```kotlin
class BankAccount {
    var balance: Double = 0.0
        private set

    fun deposit(depositAmount: Double) {
        balance += depositAmount
    }

    @Throws(InsufficientFunds::class)
    fun withdraw(withdrawAmount: Double) {
        if (balance < withdrawAmount) {
            throw InsufficientFunds()
        }

        balance -= withdrawAmount
    }

    class InsufficientFunds : Exception()

    val account = BankAccount()
    println(account.balance) // 0.0
    account.deposit(100.0)
    println(account.balance) // 100.0
    account.withdraw(50.0)
    println(account.balance) // 50.0
}
```

여기서 `BankAccount`는 계좌에 얼마 만큼의 돈이 있는지 나타내는 '상태'(`balance`)를 보유하고 있으며, 이러한 상태 유지는 양날의 검과 같다.
시간이 지남에 따라 변하는 요소들을 표현할 수 있어 유용하지만, 한편으로는 아래와 같은 이유로 상태 관리가 어려워 진다.

- 상태 변화를 '추적'해야 하기에, 많은 변경점을 갖는 프로그램은 코드를 이해하고 디버그하는 것이 어렵다. 특히 예상치 못한 오류 발생 시 해결하기 더 어려워진다.
- 가변 상태는 언제든지 변경될 수 있기 때문에 코드를 이해하고 추론하는 것을 어렵게 만든다.
- '멀티 스레드 프로그램'에서 모든 변동성은 잠재적인 충돌을 야기해 적절한 동기화가 꼭 필요하다.
- 가변 요소는 변경 가능한 모든 상태를 테스트해야 하므로 테스트하기 까다로워진다.
- 상태 변경 시, 해당 사항을 알려야 한다. 이를테면 정렬된 가변 컬렉션에 요소를 추가하면 해당 컬렉션을 재정렬 해야한다.

상태 일관성 문제와 많은 변경점은 프로젝트 복잡성이 증가되는 것은 모두 경험했을 것이다.  
아래는 공유 상태를 관리하는 것이 얼마나 어려운지에 대한 예시이다.  
여러 스레드가 동일한 속성을 수정하려고 시도하지만, 충돌로 인해 일부 작업들이 손실되는 경우를 보여준다.

```kotlin
var num = 0

for (i in 1..1000) {
    thread {
        Thread.sleep(10)
        num += 1
    }
}

Thread.sleep(5000)
println(num) // 매번 1000이 되는 것은 희박하고 대부분의 경우 다른 숫자가 나올 것임. e.g : 973
```

'Thread' 관여가 덜한 'Coroutine'을 사용하더라도 충돌이 덜 발생하지만, 여전히 발생할 수 있다.

```kotlin
suspend fun main() {
    var num = 0
    for (i in 1..1000) {
        coroutineScope {
            launch {
                delay(10)
                num += 1
            }
        }
    }

    println(num) // 매번 다른 숫자가 출력이 됨. e.g : 994
}
```

실제 프로젝트에서는 아래와 같이 적절한 동기화를 구현하여 해결한다 : `synchronized`  
하지만 적절한 동기화를 구현하는 것은 어렵고, 변경점이 많을수록 더 어려워지기에, 가변성을 제한하는 것이 좋다.

```kotlin
val lock = Any()
var num = 0

for (i in 1..1000) {
    thread {
        Thread.sleep(10)
        synchronized(lock) {
            num += 1
        }
    }
}

Thread.sleep(1000)
println(num) // 1000
```

이처럼 가변성의 단점이 너무 많아 상태 변경을 전혀 허용하지 않는 언어도 있으며 대부분 순수 함수형 언어로 'Haskell' 등이 있다.  
그러나, 이런 언어들은 가변성이 매우 제한적이어서 프로그래밍하기 매우 어렵기에 매우 드물게 사용된다.  
상태 변경은 실제 시스템의 상태를 나타내는 매우 유용한 방법이기에, 가변성을 '절제'하고 변경점을 현명하게 결정하는게 좋다.

---

## Limiting mutability in Kotlin

Kotlin은 가변성을 쉽게 제한할 수 있도록 지원하고 설계되었다.  
아래와 같은 기능과 특성으로 불변 객체를 만들거나 속성을 불변으로 유지하기 간편하다.

- read-only `val`
- 가변 컬렉션과 불변 컬렉션의 분리
- `data class`를 `copy`

---

## Read-only properties `val`

Kotlin에서는 각 속성을 다음과 같이 선언할 수 있다.

- read-only - `val` (like value)
- read-write - `var` (like variable)

`val`은 변경될 수 없다. 즉, 재할당을 허용하지 않는다.

```kotlin
val a = 10
a = 20 // Error
```

그러나, read-only 속성이 반드시 불변이거나 최종 값을 보유하고 있다는 것은 아니다.  
read-only 속성은 가변 객체를 보유할 수 있다.

```kotlin
val list = mutableListOf(1, 2, 3)
list.add(4)
println(list) // [1, 2, 3, 4]
```

또한 다른 속성에 의존하고 'custom getter'를 사용하여 정의될 수 있다.

```kotlin
var name: String = "Kotlin"
var surname: String = "Hello"
val fullName: String
    get() = "$name $surname"

fun main() {
    println(fullName) // Kotlin Hello
    name = "Java"
    println(fullName) // Java Hello
}
```

'custom getter' 값을 호출할 때마다 해당 'getter'가 호출되는 점을 알아야 한다.

```kotlin
fun calculate(): Int {
    print("Calculating...")
    return 42
}

val fizz = calculate() // Calculating...
val buzz
    get() = calculate()

fun main() {
    println(fizz) // 42
    println(buzz) // Calculating... 42
}
```

Kotlin에서 속성들은 기본적으로 캡슐화되어 있고 'custom getter/setter'를 가질 수 있다는 특징은 매우 중요하다.  
이런 특징은 개발자가 API의 변경하거나 정의할 때 유연성을 제공하기 때문이다.

한편으로, read-only `val`의 값은 변경될 수 있지만, 변경점을 제공하지 않는다.  
왜냐하면 변경점은 프로그램을 동기화 하거나 추론할 때 주요 문제의 원인이 되기 때문이다.  
이런 이유로 개발자들은 `var` 보다 `val`을 선호 한다.

그러나, `val` 속성은 '참조의 변경이 불가능' 하지만 '참조 객체의 내부 상태'는 'custom getter' 또는 'delegate'에 의해 변경될 수 있음을 기억해야 한다.
이는 앞서 말한 개발자에게 변경에 대한 더 많은 유연성을 제공한다.
그러나 변경이 필요하지 않다면 'custom getter' or 'delegate' 없이 고정된 값을 명확하게 정의하여 'final 속성'으로 사용하는 것이 좋다.

또한 Kotlin은 스마트 캐스팅을 지원하여 `val` 속성의 타입을 더 효율적으로 처리할 수 있다.

```kotlin
var name: String? = "Kotlin"
var surname: String = "Hello"

val fullName: String?
    get() = name?.let { "$it $surname" }
val fullName2: String? = name?.let { "$it $surname" }

fun main() {
    if (fullName != null) {
        println(fullName.length) // Error : smart-casted impossible
    }

    if (fullName2 != null) {
        println(fullName2.length) // Kotlin Hello
    }
}
```

`fullName`은 'custom getter'에 의해 동적으로 값이 생성되기에, 컴파일러가 `null`이 아님을 보장할 수 없어 스마트 캐스팅을 사용할 수 없다.
반대로, `fullName2`는 초기화 시점에 정적으로 값이 생성되기에, 컴파일러가 `null`이 아님을 보장할 수 있어 스마트 캐스팅을 사용할 수 있다.

## Separation between mutable and read-only collections

Kotlin에서 컬렉션의 계층 구조가 아래와 같이 설계된 방식 덕분에 속성과 마찬가지로 'read-only 컬렉션'을 지원한다.

<img src="CollectionHierarchy.png" width="60%">

'Iterable', 'Collection', 'Set', 'List' 인터페이스는 'read-only' 타입으로, 수정을 허용하는 메서드가 없다.
반면에, 'MutableIterable', 'MutableCollection', 'MutableSet', 'MutableList' 인터페이스는 'read-only' 인터페이스를 확장하고, 변경을 허용하는 메서드를
추가한다.

이는 속성이 작동하는 방식과 유사하며, 'read-only 속성'은 'getter'만을 포함하는 반면, 'read-write 속성'은 'getter/setter'를 포함한다.

Kotlin에서 'read-only 컬렉션'은 내부적으로 변경 가능하지만, 'read-only interface'를 통해 외부에 노출되기에 외부에서 변경할 수 없다.   
예를 들어, `Iterable<T>.map`과 `Iterable<T>.filter` 함수는 내부적으로 `ArrayList`를 사용하여 작업을 수행하지만,
결과는 'read-only 컬렉션'인 `List`로 반환하여 외부에서 변경할 수 없도록 한다.

```kotlin
inline fun <T, R> Iterable<T>.map(
    transform: (T) -> R
): List<R> {
    val list = ArrayList<R>()
    for (item in this@map) {
        list.add(transform(item))
    }
    return list
}
```

이와 같이 컬렉션 인터페이스를 완전한 불변이 아닌 'read-only'로 설계하는 것은 개발자에게 많은 유연성을 제공한다.  
실제로, 'read-only 컬렉션'을 구현하는 실제 컬렉션을 다양하게 사용 할 수 있다. 이로 인해, 개발자는 특정 플랫폼에 맞는 컬렉션을 유연하게 사용할 수 있다.
예를 들어, 메모리 사용이 중요한 모바일 앱의 경우 `ArrayList`를, 빠른 Insert/Delete가 중요한 서버 사이드는 `LinkedList`를 선택할 수 있다.

Kotlin에서 'read-only 컬렉션'을 사용하는 것은 불변 컬렉션을 사용하는 것과 거의 동일한 수준의 안전성을 제공한다.  
'read-only 컬렉션'의 유일한 위험은 개발자가 의도적으로 다운 캐스팅(down-casting)을 수행하여 컬렉션의 불변성을 우회하려고 시도할 떄 발생된다.
Kotlin에서 'read-only 컬렉션'을 반환하는 것은 'contract'의 일부로, 개발자는 반환된 컬렉션이 오직 'read' 목적으로만 사용될 것이라는 점을 신뢰해야 한다.
만약 이러한 'contract'를 어길 경우 프로그램의 안정성이 보장되지 않는다.

아래와 같은 다운 캐스팅은 추상화 대신 구체적인 구현에 의존하게 만든다.  
이는 코드의가독성과 유지 보수성을 저하시키고, 불안전하며 예상치 못한 결과를 초래할 수 있다.

```kotlin
val list = listOf(1, 2, 3)

// Don't Do this
if (list is MutableList) {
    list.add(4)
}
```

Kotlin에서 `listOf`는 플랫폼 별로 다른 구현체를 반환할 수 있다.  
예를 들어, JVM 환경에서 `listOf`는 `Arrays.ArrayList`를 반환한다.   
이는 Java의 `List` 인터페이스를 구현하며, `add`와 `set`과 같은 메서드를 포함한다.

Java의 `List`는 Kotlin의 `MutableList`에 해당된다. 즉, Java의 `List`를 사용할 때 Kotlin에서는 `MutableList` 처럼 다룰 수 있다.  
그러나 `Arrays.ArrayList`와 같은 특정 구현체는 `MutableList`의 모든 연산을 지원하지 않을 수 있기에 주의해야 한다.

Kotlin에서 'read-only 컬렉션'을 가변 컬렉션으로 다운 캐스팅하는 것은 금지되어 있다.  
만약 'read-only 컬렉션'에서 가변 컬렉션으로 변경해야 할 필요가 있다면, `List.toMutableList`를 사용하여 복사본을 생성해야 한다.

```kotlin
val list = listOf(1, 2, 3)

val mutableList = list.toMutableList()
mutableList.add(4)
```

## Copy in data classes

`String`or`Int`와 같이 내부 상태가 변하지 않는 불변 객체를 선호하는 이유는 다음과 같다.

1. 한 번 생성된 후 상태가 변경되지 않기에 프로그램의 행동을 예측하고 이해하기 더 쉽다.
2. 불변 객체는 공유되어도 상태가 변하지 않아, 병렬 프로그래밍 시 데이터 경쟁이나 충돌의 위험이 없다.
3. 상태가 변경되지 않기에, 불변 객체의 참조를 안전하게 캐싱할 수 있다.
4. 복사할 때 원본 객체의 상태가 변경될 위험이 없기에, 'defensive copy', 'deep copy'를 할 필요가 없다.
5. 불변 객체를 기반으로 다른 객체를 구성할 때 이상적이며 작업하기 더 쉽다. 또한 변경성을 쉽게 제어할 수 있다.
6. `Set`에 추가하거나 `Map`의 'Key'로 안정적으로 사용할 수 있다.

```kotlin
val names: SortedSet<FullName> = TreeSet()
val person = FullName("AAA", "AAA")
names.add(person)
names.add(FullName("BBB", "BBB"))
names.add(FullName("CCC", "CCC"))

print(names) // [AAA AAA, BBB BBB, CCC CCC]
print(person in names) // true

person.name = "ZZZ"
print(names) // [ZZZ AAA, BBB BBB, CCC CCC]
print(person in names) // false, because person is at incorrect position
```

가변 객체는 데이터가 변경될 수 있어 예측하기 어렵고 위험할 수 있는 반면,
불변 객체는 한 번 생성되면 상태가 변경되지 않아 안정적이지만, 때로는 데이터 변경이 필요할 수 있다.
이러한 데이터 변경에서는 불변 객체는 객체 자체의 변경이 아닌, 데이터 변경 후의 새로운 객체를 생성하는 메서드를 가져야 한다.

예를 들어, `Int`는 불변으로, `plus`나 `minus` 같은 메서드를 호출할 때, 원래의 `Int`를 수정하지 않고 해당 연산 후 새로운 `Int`를 반환한다.
이는 `Iterable`과 같은 'read-only' 객체에도 적용되며, 컬렉션 처리 함수들은 원본을 변경하지 않고 새로운 컬렉션을 반환한다.

위와 같은 원칙을 불변 객체에 적용 할 수 있으며, 아래 예시를 보자.
불변 클래스 `User`가 있고, 그 `surname`을 변경할 수 있어야 한다고 가정해보면,
특성 속성을 변경 후 복사본을 생성하는 메서드를 통해 이를 지원할 수 있다.

```kotlin
class User(
    val name: String,
    val surname: String
) {
    fun withSurname(surname: String): User = User(name, surname)
}

// usage
var user = User("yongsuk", "park")
user = user.withSurname("kim")
print(user) // User(name=yongsuk, surname=kim)
```

위와 같이 메서드를 작성하여 불변 객체에서 각 속성을 변경하도록 할 수 있지만, 모든 속성에 대해 이를 수행하는 것은 번거로운 작업이 될 수 있다.  
이를 해결하기 위해 'data modifier'가 제공하는 메서드 중 하나인 `copy`를 사용할 수 있다.

`copy`는 객체의 모든 'primary constructor' 속성을 기본값으로 사용하여 새로운 인스턴스를 생성한다.  
즉, 기존 객체의 상태를 기본값으로 하고, 변경하고자 하는 특정 속성만 새로운 값으로 지정할 수 있다.  
이렇게 함으로써, 기존 객체는 그대로 유지되며, 변경된 속성을 가진 새로운 객체를 생성할 수 있다.

```kotlin
data class User(
    val name: String,
    val surname: String
)

// usage
var user = User("yongsuk", "park")
user = user.copy(surname = "kim")
print(user) // User(name=yongsuk, surname=kim)
```

이런 방법은 가변 객체를 직접 사용하는 것보다 효율적이지 않을 수 있지만,
불변 객체의 모든 장점을 가지고 있어 일반적인 상황에서 가변 객체 보다 더 선호되어야 한다.

---

## Different kinds of mutation points

변경이 가능한 목록이 필요하다고 가정하면, 가변 컬렉션을 사용하는 방법과 'read-write' `var`을 사용하는 방법으로 나눌 수 있다.

```kotln
val list1: MutableList<Int> = mutableListOf()
var list2: List<Int> = listOf()
```

위 방법 모두 속성을 수정할 수 있지만, 각각 다른 방식으로 수정된다.

```kotlin
list1.add(1)
list2 = list2 + 1
```

위와 같이 모두 'plus-assign' 연산자를 사용하여 대체될 수 있지만, 각각 다른 행동으로 반환된다.

```kotlin
list1 += 1 // Translates to list1.plusAssign(1)
list2 += 1 // Translates to list2 = list2.plus(1)
```

위 두 방법 모두 올바르며 각각 장단점이 있다.

가변 컬렉션의 경우 리스트 구현체에서 변화가 일어난다. 이 방법의 장점은 구현이 직관적이고 쉽다.  
하지만 멀티스레드 환경에서 컬렉션 구현에 따라 동기화 여부가 달라져 스레드 안전성을 보장하지 않는다.  
따라서 멀티 스레드 환경에서 가변 컬렉션 사용 시 직접 동기화 메커니즘을 구현해야 한다.

`var`의 경우, 변경점은 해당 속성 자체가 된다. 이 방법의 장점은 전체적인 보안성이 더 높다.  
단일 변경점을 가지고 있기에 해당 속성에 대한 접근을 제어함으로써 동기화를 보다 쉽게 관리할 수 있다.   
하지만 이 경우에도 멀티 스레드 환경에서 동기화 메커니즘 없이 `var` 속성을 변경하는 경우
데이터 일관성 문제나 요소의 손실과 같은 문제가 발생할 수 있기에 동기화 메커니즘을 직접 구현해야 한다.

```kotlin
var list = listOf<Int>()
for (i in 1..1000) {
    thread {
        list = list + i
    }
}

Thread.sleep(1000)
print(list.size) // 매번 1000이 되는 것은 희박하고 대부분의 경우 다른 숫자가 나올 것임. e.g : 944
```

가변 컬렉션 대신 `var` 속성을 사용하면, 'custom setter' 또는 'Delegate'를 정의함으로써 속성의 변경을 추적할 수 있다.  
예를 들어, 아래와 같이 'observable delegate'를 사용할 때 리스트의 모든 변경 사항을 기록할 수 있다.

```kotlin
var names by Delegates.observable(listOf<String>()) { _, old, new ->
    println("Names changed from $old to $new")
}

names += "Yongsuk" // Names changed from [] to [Yongsuk]
names += "Minsu" // Names changed from [Yongsuk] to [Yongsuk, Minsu]
```

가변 컬렉션을 사용하여 컬렉션 요소를 직접 조작하는 것은 처리 속도가 빠를 수 있지만, 컬렉션의 변경 사항을 추적하기 위해서 특별한 'observable' 구현이 필요하다.
반면 `var` 속성은 'custom setter' or 'observable delegate'를 사용하여 객체의 변경을 더 세밀하게 추적하고 제어할 수 있게 된다.

`var` 속성과 가변 컬렉션을 동시에 사용하는 것은 추천되지 않는다.

```kotlin
// Don't do that
var list3 = mutableListOf<Int>()
```

속성 자체의 변경과 컬렉션 내부의 변경, 이 두 가지 방식으로 발생할 수 있는 변화를 모두 동기화해야 하며, 'plus-assign'을 사용한 변경을 사용할 수 없다.

상태를 변경하는 모든 방식은 비용이 되기에 일반적으로 불필요한 상태 변경 가능성을 만들지 않는 것이 좋다.  
즉, 가변성을 제한하는 것이 좋다.

---

## Do not leak mutation points

상태를 구성하는 가변 객체를 외부에 노출시키는 것은 위험한 상황으로, 아래 예제를 보자.

```kotlin
data class User(val name: String)

class UserRepository {
    private val users: MutableMap<Int, String> = mutableMapOf()

    fun loadAll(): MutableMap<Int, String> = users
}
```

누군가 `loadAll`을 사용하여 `UserRepository`의 비공개 상태를 수정할 수 있는 상황이 발생할 수 있다.

```kotlin
val userRepository = UserRepository()

val users = userRepository.loadAll()
users[3] = "Kotlin"

// ...

println(userRepository.loadAll()) // {3=Kotlin}
```

특히나, 위와 같은 상태 수정이 우연하게 발생하는 경우 더욱 위험하며 예상치 못한 오류를 야기할 수 있다.  
이를 방지하기 위해 아래 2가지 방법을 사용할 수 있다.

첫 번째 방법으로는 'Defensive copying(반환된 가변 객체를 복사)'를 사용하는 것이다.

```kotlin
class UserHolder {
    private val user: MutableUser()
    fun get(): MutableUser = user.copy()
}
```

그러나 가능한 가변성을 제한하는 것을 선호하기에,
컬렉션의 경우 이런 객체들을 'read-only' 상위 타입으로 업 캐스팅하여 가변성을 제한 할 수 있다.

```kotlin
data class User(val name: String)

class UserRepository {
    private val users: MutableMap<Int, String> = mutableMapOf()

    fun loadAll(): Map<Int, String> = users
}
```