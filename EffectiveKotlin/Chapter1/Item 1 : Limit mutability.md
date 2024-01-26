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
이는 앞서 말한 개발자에게 변경에 대한 더 많은 유연성을 제공한다. 그러나 변경이 필요하지 않다면 `final` 속성을 사용하는 것이 좋다.  
`final` 속성은 그 값이 변경 될 수 없고, 값이 명확하게 정의되어 있어 추론하기 쉽다.

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

Kotlin에서 컬렉션의 계층 구주와 아래와 같이 설계된 방식 덕분에 속성과 마찬가지로 'read-only 컬렉션'을 지원한다.

<img src="CollectionHierarchy.png" width="60%">

'Iterable', 'Collection', 'Set', 'List' 인터페이스는 'read-only' 타입으로, 수정을 허용하는 메서드가 없다. 
반면에, 'MutableIterable', 'MutableCollection', 'MutableSet', 'MutableList' 인터페이스는 'read-only' 인터페이스를 확장하고, 변경을 허용하는 메서드를 추가한다.

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
val list = listOf(1,2,3)

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
val list = listOf(1,2,3)

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



---

## 다른 종류의 변경 가능 지점

변경 가능한 목록을 만들어야 한다고 가정할 때 2가지 방법으로 나타낼 수 있다.

```kotln
val list1: MutableList<Int = mutableListOf()
var list2: List<Int> = listOf()
```

이 때 2가지 모두 속성을 수정할 수 있지만 방법이 다르다.

```kotlin
list1.add(1)
list2 = list2 + 1
```

내부적으로 처리되는 방법은 다음과 같이 동작된다.

```kotlin
list1 += 1 // list1.plusAssign(1)
list2 += 1 // list2.plus(1)
```

위 2가지 방법 모두 동일하게 동작하지만 변경 가능 지점이 다르다.

- `mutableList` : 구체적인 리스트 구현 내부에 변경 가능 지점이 있어 내부적으로 적절한 동기화 여부를 확신할 수 없다.
- `var properties` : 변경 가능 지점이 `properties` 자체이기에 멀티 스레드 처리 안정성이 좋다.

> `var properties`에 `mutable collection`을 사용하지 말자.
> 만약 사용하게 될 경우 2가지 지점에 대한 동기화를 구현해야 하고 모호성으로 인해 `+=` 연산자를 사용할 수 없게 된다.

---

## 변경 가능 지점을 유출하지 말자

- `mutable object`를 외부에 노출할 경우 문제가 발생될 수 있다.

```kotlin
data class User(val name: String)

class UserRepository {
    private val users: MutableMap<Int, String> = mutableMapOf()

    fun loadAll(): MutableMap<Int, String> = users
}

val userRepository = UserRepository()
val users = userRepository.loadAll()
users[3] = "Kotlin"

// ...

println(userRepository.loadAll()) // {3=Kotlin}
```

위와 같은 수정이 발생되는 것을 방지하기 위해 2가지 방법을 사용할 수 있다.

### 방어적 복제

```kotlin
class UserHolder {
    private val user: MutableUser()
    fun get(): MutableUser = user.copy()
}
```

### 반환 타입을 immutable type으로 Up-Casting

```kotlin
data class User(val name: String)

class UserRepository {
    private val users: MutableMap<Int, String> = mutableMapOf()

    fun loadAll(): Map<Int, String> = users
}
```

---

# 정리

- `var`보다 `val`을 선호해라
- `mutable` 보다 `immutable`을 선호해라
- 변경이 필요한 경우 `immutable data class`를 만들고 `copy`를 사용해라
- `상태`를 유지하라면 `mutable collection`보다 `immutable collection`을 사용해라
- 변경점을 신중하게 설계하고 불필요한 변경점을 만들지마라
- `mutable object`를 외부에 노출하지 마라