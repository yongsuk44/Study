# Item 46: 함수 타입 파라미터가 있는 함수에 inline 키워드 사용하기 

Kotlin 표준 라이브러리에서 대부분의 고차(higher-order) 함수에 `inline` 수식어가 정의되어 있는 것을 알 수 있습니다.
아래는 `repeat` 함수의 구현입니다.

```kotlin
inline fun repeat(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}
```

`inline` 수식어는 컴파일 도중 이 함수의 모든 사용을 함수 본문으로 대체합니다. 
또한, `repeat` 내부의 함수 인자에 대한 모든 호출도 그 함수들의 본문으로 대체됩니다. 

따라서 다음 `repeat(10) { print(it) } `은 컴파일 도중 다음과 같이 대체됩니다. `for (index in 0 until 10) { print(index) }`

`inline` 수식어는 함수가 실행되는 방식에 비해 상당한 변화를 가져옵니다. 
일반적인 함수에서는 실행이 함수 본문으로 점프하고 모든 구문을 호출한 다음, 함수가 호출된 위치로 다시 돌아옵니다.
하지만 호출을 본문으로 바꾸는 것은 상당히 다른 동작입니다. 이러한 동작 방식에는 아래와 같은 이점이 있습니다. 

### 타입 인자가 실체화될 수 있습니다. 

이는 제네릭 타입에 대한 정보를 유지할 수 있다는 의미입니다.
일반적으로 제네릭 함수는 컴파일 시점에 타입 인자 정보를 잃어버리는데 `inline` 함수를 사용하면 이를 피할 수 있습니다.

실체화된 타입 인자는 컴파일 시점에도 그 정보를 유지하기 때문에, 제네릭 타입의 클래스를 검사하는 등의 작업이 가능합니다.

```kotlin
// 제네릭 타입에 대한 정보를 검사하려 했을 때 컴파일 시점에서 "Cannot check for instance of erased type : T" 오류 발생
fun <T> checkType(value: T) {
    if (value is String) println("It is a String")
}

// 가능
inline fun <reified T> checkType(value: Any) {
    if (value is T) println("It is a ${T::class.simpleName}")
}
```

### 함수 파라미터가 함수 일 때 `inline` 함수가 더 빠릅니다.

`inline` 함수는 호출되는 시점에서 함수 본문으로 대체되기 때문에, 별도의 함수 호출 비용이 없습니다. 따라서 함수를 파라미터로 전달할 때 발생하는 오버헤드가 없습니다.

```kotlin
// 호출할 때마다 새로운 함수 객체가 생성되어 메모리와 성능에 영향을 줄 수 있음
fun performOperation(value: Int, operation: (Int) -> Int): Int {
    return operation(value)
}

// 함수를 호출 시점에 `{ it * 2 }` 본문으로 대체되어 함수 객체 생성 오버헤드가 없음
inline fun performOperation(value: Int, operation: (Int) -> Int): Int {
    return operation(value)
}

val result = performOperation(
    value = 5,
    operation = { it * 2 }
)
```

### Non-local `return`이 허용됩니다.

Non-Local 반환이 허용되는것은 `inline` 함수를 사용할 때 일반 함수와는 달리 내부 함수에서도 외부 함수를 벗어날 수 있습니다. 이를 통해 코드의 가독성과 흐름 제어를 향상 시킬 수 있습니다.

```kotlin
// `return` 구문은 `forEach` 람다 함수를 벗어나지만 `printValues` 함수를 벗어나지는 않음
fun printValue(values: List<Int>) {
    values.forEach { value ->
        if (value > 4) return
        println("$value")
    }
}

// `inline` 사용 시 `return` 구문은 `printValue` 함수를 완전히 벗어나게 되어 코드의 흐름을 더 자유롭게 제어할 수 있음
inline fun printValue(values: List<Int>) {
    values.forEach { value ->
        if (value > 4) return
        println("$value")
    }
}

printValue(listOf(1, 2, 3, 4, 5, 6, 7)) // print : 1 2 3 4
```

---

## `reified` 타입 인자와 `inline` 함수 활용

제네릭은 2004년 J2SE 5.0 버전에서 Java 프로그래밍 언어에서 추가되었습니다.

그러나 JVM 바이트 코드에는 여전히 포함되지 않아서 컴파일 중에 제네릭 타입은 지워집니다.

예를 들어 `List<Int>`는 `List`로 컴파일됩니다. 이 때문에 객체가 `List<Int>` 인지 확인할 수 없고 `List` 인지만 확인할 수 있었습니다.

```kotlin
any is List<Int> // Error
any is List<*> // OK
```

위와 같은 이유로 아래와 같이 타입 인자에 연산을 수행할 수 없습니다.

```kotlin
fun <T> printTypeName() {
    print(T::class.simpleName) // Error
}
```

이러한 제한을 `inline`과 `reified` 수식어를 사용하면,
컴파일 타임에 타입 정보가 지워지는 것을 방지하고 런타임에도 이 정보를 활용할 수 있게 됩니다.

`inline` 함수 호출이 그 본문으로 대체되고, `reified` 수식어를 사용하여 타입 파라미터의 사용을 타입 인자로 대체할 수 있습니다.

```kotlin
inline fun <reified T> printTypeName() {
    print(T::class.simpleName) // OK
}

printTypeName<Int>() // Int
printTypeName<Char>() // Char
printTypeName<String>() // String
```

컴파일 중 위 내용이 아래와 같이 타입 인자가 타입 파라미터를 대체합니다.

```kotlin
print(Int::class.simpleName) // Int
print(Char::class.simpleName) // Char
print(String::class.simpleName) // String
```

`reified`는 유용한 수식어로 `inline` 함수와 함께 사용됩니다. 예를 들어, `filterIsInstance`는 특정 타입의 요소만 필터링하는 데 사용됩니다.

```kotlin
class Worker
class Manager

val employees: List<Any> = listOf(Worker(), Manager(), Worker())

val workers: List<Worker> = employees.filterIsInstance<Worker>()
```

---

## 함수형 파라미터를 가진 함수는 `inline` 처리될 떄 더 빠르게 동작함

보다 구체적으로는 모든 함수는 `inline` 처리될 때 약간 더 빠릅니다.
실행 중 점프를 할 필요가 없고, 백스택을 추적할 필요가 없기 때문입니다.

위와 같은 이유로 Kotlin 표준 라이브러리에서 자주 사용되는 작은 함수들이 대부분 `inline` 처리되는 이유 입니다.

그러나 아래와 같이 함수가 함수형 파라미터를 가지지않다면, 이 차이는 크게 중요하지 않습니다. 

```kotlin
inline fun print(message: Any?) {
    System.out.print(message)
}
```

함수는 객체로서 동작할 수 있습니다.
Kotlin에서 함수는 특수한 종류의 객체로 취급되며 이들은 함수 리터럴(function literals)을 사용해 생성되며, 함수형 프로그래밍에서 이 개념은 '함수가 일급 시민'임 의미합니다.

즉 함수를 변수로 저장하거나, 다른 함수의 인자로 전달하거나, 함수의 결과로 반환할 수 있습니다.

이러한 특성은 코드의 재사용성을 높이고 코드를 간결하고 이해하기 쉽게 만들 수 있습니다.

함수를 객체로 취급할 때, 이 객체를 메모리에 유지해야 합니다. 이는 JS와 같은 언어에서는 간단하지만 Kotlin/JVM에서는 좀 더 복잡한 방식을 사용해야 합니다.

```kotlin
val lambda: () -> Unit { 
    // code
}
```

위 코드는 람다 함수를 정의하는 코드로 이는 객체로 취급되므로 JVM에서는 이를 저장하기 위한 객체가 필요합니다.

이러한 객체는 JVM의 익명 클래스를 사용하거나 일반 클래스를 사용해서 생성될 수 있습니다.

따라서 위 람다 함수는 아래와 같이 변환될 수 있습니다.

```java
// Java
Function0<Unit> lambda = new Function0<Unit>() {
    public Unit inovke() {
        // code
    }
}
```

또는 별도의 파일에 정의된 일반 클래스로 변활될 수 있습니다.

```java
// Java
// Additional class in separate file

public class Test$lambda implements Function0<Unit> {
    public Unit invoke() {
        // code
    }
}

Function0 lambda = new Test$lambda()
```

함수의 타입이 JVM에 컴파일될 때 각 타입은 특정 인터페이스로 변환됩니다.

- `() -> Unit` -> `Function0<Unit>`
- `() -> Int` -> `Function0<Int>`
- `(Int) -> Int` -> `Function1<Int, Int>`
- `(Int, Int) -> Int` -> `Function2<Int, Int, Int>`

함수 타입의 구현은 Kotlin 컴파일러가 생성하는 인터페이스를 통해 구현됩니다.
즉, 인터페이스는 요구에 따라 생성되므로 명시적으로 사용할 수 없으며 대신 함수 타입을 사용해야 합니다.

예를 들어 아래와 같은 클래스 `OnClickListener`는 함수 타입 `() -> Unit`을 구현하며, 이는 Kotlin 컴파일러가 생성한 인터페이스를 구현한 것 입니다.

```kotlin
class OnClickListener: () -> Unit {
    override fun invoke() {
        // ...
    }
}
```

함수의 본문을 객체로 감싸면 코드 실행이 느려집니다. 이는 함수를 실행할 때마다 새로운 객체가 생성되기 때문입니다.

아래 첫번째 함수는 `inline` 함수로 함수 호출을 통해 코드가 복사되어 코드내에 직접 삽입됩니다.
따라서 객체 생성과 관련된 오버헤드가 없습니다. 반면에 두번째 함수는 `inline`이 아니므로 객체 생성과 함수 호출에 관련된 오버헤드가 발생됩니다.

이러한 이유로 아래 두 함수 중 첫 번째 함수가 더 빠릅니다.

```kotlin
inline fun repeatInline(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}

fun repeatNoinline(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}
```

위와 같은 차이는 실제 코드에서 그다지 중요하지 않을 수 있습니다.
그러나 테스트를 잘 설계하면 이러한 차이를 명확하게 확인할 수 있습니다.

아래 코드 예시를 보면 첫번째 함수는 평균 189ms, 두번째 함수는 평균 447ms가 소요됩니다.

이 차이는 첫번째 함수는 숫자를 반복하고 빈 함수를 호출하는 반면, 두번째 함수는 숫자를 반복하고 객체를 호출하고, 이 객체는 빈 함수를 호출하기 때문입니다.

```kotlin
@Benchmark
fun nothingInline(blackHole: BlackHole) {
    repeatNoinline(100_000_000) { blackHole.consume(it) }
}

@Benchmark
fun nothingNoninline(blackHole: BlackHole) {
    repeatInline(100_000_000) { blackHole.consume(it) }
}
```

`inline` 함수와 `inline`이 아닌 함수의 더 큰 차이는 함수 리터럴에서 지역 변수를 캡쳐할 때 나타납니다.

캡쳐된 값은 객체로 감싸져야 하며 사용될 때마다 이 객체를 사용해야 합니다.

```kotlin
var a = 2L
noinlinerepeat(100_000_000) { a += a / 2 }
```

위 코드에서 변수 `a`는 람다 함수에 캡처되고 있습니다. 

이는 `inline` 함수가 아닌 람다에서 직접 사용될 수 없으므로, 컴파일 시점에 `a`의 값은 참조 객체로 감싸집니다.


```kotlin
val a = Ref.LongRef() 
a.element = 2L
noinlineRepeat(100_000_000) { a.element += a.element / 2 }
```

이렇게 되면, 객체는 매우 빈번하게 사용될 수 있으며, 이는 함수가 매번 호출될 때마다 발생됩니다.

이로 인해, `inline` 함수와 `inline`이 아닌 함수 사이에는 매우 큰 성능 차이가 발생할 수 있습니다.

또한 아래 코드를 보게되면 첫번째 함수는 0.2ns 만에 실행되지만 두 번째 함수는 283,000,000ns가 걸립니다.
이는 함수가 객체로 컴파일되고, 로컬 변수가 감싸져야 하기 때문입니다.

```kotlin
fun repeatInline() {
    var a = 2L
    repeat(100_000_000) { a += a / 2 }
}

fun repeatNoninline() {
    var a = 2L
    noinlineRepeat(100_000_000) { a += a / 2 }
}
```

함수를 `inline`으로 선언하면, 위와 같은 오버헤드를 최소화할 수 있습니다.

특히 함수형 파라미터를 가진 유틸리티 함수를 정의할 때는 일반적으로 그 함수를 `inline`으로 만드는 것이 좋습니다.

---

## Non-local return 허용

이전에 정의한 `repeatNoninline`의 구현 부분은 아래 `if`또는 `for`와 매우 비슷하게 구현되어 있습니다.

```kotlin
if(value != null) print(value)

for (i in 1..10) print(i)

repeatNoninline(10) { print(it) }
```

그러나 한 가지 중요한 차이점은 `return`이 내부에 허용되지 않습니다.

```kotlin
fun main() {
    repeatNoninline(10) {
        print(it)
        return // Error : Not allowed
    }
}
```

이는 함수 리터럴이 어떻게 컴파일 되는지에 따라 달라지게 됩니다. 

만약 함수 리터럴이 인라인 되지 않으면 코드는 다른 클래스에 위치해 있다고 판단하여 `main`에서 `return`할 수 없습니다.  
그러나 함수 리터럴이 인라인 될 때 코드는 어차피 `main` 함수에 위치하게 되기 때문에 `return`을 허용할 수 있게 됩니다.

```kotlin
fun main() {
    repeatInline(10) {
        print(it)
        return // OK
    }
}
```

덕분에 함수는 제어 구조처럼 보이고 동작할 수 있습니다.

```kotlin
fun getSomeMoney(): Money? {
    repeat(100) {
        val money = searchForMoney()
        if (money != null) return money
    }
    return null
}
```

---

## inline 수식어 비용 

`inline`은 유용한 수식어이지만 어디서나 사용될 수 있는 것은 아닙니다.
`inline` 함수는 재귀로 사용할 수 없으며 반복적인 사이클은 더 위험합니다.

만약 재귀로 사용할 경우 자기 자신을 무한하게 호출하게 될 것이며, 반복적인 사이클 형태로 만드는 경우  컴파일 시간에 에러를 발생시키지 않고, 런타임에 에러를 발생시키게 됩니다.

```kotlin
inline fun a() { b() }
inline fun b() { c() }
inline fun c() { a() }
```

`inline` 함수는 가시성을 제한한 요소를 사용할 수 없습니다.

```kotlin
internal inline fun read() {
    val reader = Reader() // Error
    // ...
}

private class Reader { /* ... */ }
```

---

## Crossinline과 noinline

함수를 `inline` 처리하고 싶지만 어떠한 이유로 모든 함수 타입 인수를 `inline` 할 수 없는 경우가 필요할 수 있습니다. 

이러한 경우 다음 수식어를 사용하여 처리할 수 있습니다.

| 타입 | 설명                                                       |
| --- |----------------------------------------------------------|
| `crossinline` | 함수가 `inline` 되어야 하지만 'non-local return'이 허용되지 않음을 의미합니다. |
| `noinline` | `inline` 함수의 함수 타입 인수가 `inline` 되지 않아야 함을 의미합니다.   |

```kotlin
inline fun requestNewToken(
    hasToken: Boolean, 
    crossinline onRefresh: () -> Unit,
    noinline onGenerate: () -> Unit
) {
    if (hasToken) {
        // 함수가 inline 되지 않은 함수에 인수로 함수를 전달해야 하므로 noinline을 사용해야 합니다.
        httpCall("get-token", onGenerate)
    } else {
        httpCall("refresh-token") {
            // 'non-local return'이 허용되지 않는 컨텍스트에서 함수를 인라인하려면 crossinline을 사용해야 합니다.
            onRefresh()
            onGenerate()
        }
    }
}

fun httpCall(url: String, callback: () -> Unit) { /* ... */ }
```