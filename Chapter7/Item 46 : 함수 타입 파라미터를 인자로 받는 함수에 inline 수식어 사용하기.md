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