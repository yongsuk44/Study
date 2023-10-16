# Scope functions

### let

> - 객체 참조는 인자(`it`) 사용, 반환 값은 람다의 결과
> - 연속적인 연산의 결과를 변수에 할당하지 않고 처리하는 경우에 주로 사용
> - `?.` 연산자와 함께 사용하여 `null` 검사를 자동으로 수행할 때 사용

`let`은 주로 연속적인 연산의 결과를 변수에 할당하지 않고 처리할 경우에 사용됩니다.  
예를 들어 아래 코드는 컬렉션에 대한 2번의 연산 결과를 출력합니다.

```kotlin
val numbers = mutableListOf("one", "two", "three", "four", "five")
val resultList = numbers.map { it.length }.filter { it > 3 }
println(resultList)
```

이 떄 `let`을 사용하면 컬렉션에 대한 연산 결과를 별도의 변수에 할당하지 않아도 됩니다:

```kotlin
val numbers = mutableListOf("one", "two", "three", "four", "five")
numbers
    .map { it.length }
    .filter { it > 3 }
    .let { 
        println(it)
        // and more function calss if needed
    }
```

`let`에 전달된 코드 블록에서 단일 함수만 호출하고, 그 함수의 인자로 `it`을 사용한다면 메서드 참조(`::`)를 사용할 수 있습니다. 

```kotlin
val numbers = mutableListOf("one", "two", "three", "four", "five")
numbers.map { it.length }.filter { it > 3 }.let(::println)
```

`let`은 종종 `null`이 아닌 값에 대한 연산을 SafeCallOperator(`?.`)와 함께 사용하여 `null` 검사를 자동으로 수행할 때 사용됩니다.  
이 때 `let` 블록 내에서 객체 참조 `it`은 `non-null`이므로, 안전하게 처리할 수 있습니다.

```kotlin
val str: String? = "Hello"
// processNonNullString(str) // compilation error: str can be null
val length = str?.let { 
    println("let() called on $it")
    processNonNullSting(it) // OK: 'it' is not null inside '?.let { }'
    it.length
}
```

`let` 함수 내부에서 람다 인자의 이름을 지정하여 컨텍스트 객체에 대한 새로운 로컬 변수를 생성할 수 있습니다.  
이렇게 하면, 코드 블록 내에서 생성된 변수를 `it` 대신 사용할 수 있어 코드 가독성이 향상됩니다.

```kotlin
val numbers = listOf("one", "two", "three", "four")
val modifiedFirstItem = numbers.first().let { firstItem ->
    println("The first item of the list is '$firstItem'")
    if (firstItem.length >= 5) firstItem else "!$firstItem!"
}.uppercase()

println("First item after modifications : '$modifiedFirstItem'")
```

---

### with

> - 객체 참조는 리시버(`this`) 사용, 반환 값은 람다의 결과 
> - 컨텍스트 객체에 대한 여러 작업을 묶어 코드의 가독성을 높이는 경우에 주로 사용
> - 일반적인 가이드라인은 반환 값을 사용할 필요가 없을 때 `with` 사용, 반환 값을 활용해도 무방함

`with` 함수 람다 블록에서 컨텍스트 객체를 `this`로 사용하며, `this`를 통해 컨텍스트 객체의 속성이나 메서드에 직접 접근할 수 있습니다.

`with` 블록의 마지막 표현식의 결과가 `with` 함수의 반환 값이 되지만, **일반적으로 `with`은 반환 값을 사용하지 않을 때 주로 사용됩니다.**  
또한 컨텍스트 객체에 대한 여러 작업을 묶을 때 유용하므로, 코드의 가독성을 높이는 데 도움이 됩니다. 

```kotlin
val numbers = mutableListOf("one", "two", "three")
with(numbers) {
    println("'with' is called with argument $this")
    println("It contains $size elements")
}
```

`with`을 사용하여 속성이나 함수가 값 계산에 사용되는 도우미 객체를 도입할 수도 있습니다.

`with` 함수는 컨텍스트 객체를 리시버(`this`)로 사용하므로, 컨텍스트 객체의 속성이나 메서드를 직접 호출할 수 있습니다.  
이런식으로 여러 속성이나 함수를 한 번에 쉽게 사용할 수 있어 코드 가독성과 유지보수성이 향상됩니다.

```kotlin
val numbers = mutableListOf("one", "two", "three")
val firstAndLast = with(numbers) { 
    "The first element is ${first()}, the last element is ${last()}"
}
println(firstAndLast)
```

---

### run

> - 객체 참조는 리시버(`this`)로 사용, 반환 값은 람다의 결과
> - 객체의 속성을 초기화하고 동시에 어떤 값을 계산하고 반환할 때 주로 사용
> - 컨텍스트 객체의 확장 함수 또는 컨텍스트 객체가 없는 비확장 함수로 사용 가능

`run`은 `with`과 같은 기능을 하지만, 확장 함수로 구현되어 있어 컨텍스트 객체에서 `.` 표기법을 사용하여 호출할 수 있습니다.  
람다 블록내에서 `this`를 사용하여 컨텍스트 객체의 속성과 메서드에 접근할 수 있습니다.

또한 람다 블록의 마지막 표현식이 `run`의 반환 값이 됩니다.

```kotlin
val service = MultiportService("https://example.kotlinlang.org", 80)
val result = service.run {
    port = 8080
    query(prepareRequest() + " to port $port")
}

// the same code written with let() function:
val letResult = service.let {
    it.port = 8080
    it.query(it.prepareRequest() + " to port ${it.port}")
}
```

`run`을 비확장 함수로도 호출할 수 있습니다. 비확장 버전의 `run`은 컨텍스트 객체가 없으며 람다 블록의 결과를 반환합니다.

비확장 `run`을 사용하면 하나의 표현식 위치에서 여러 문장을 실행할 수 있습니다. 예를 들어 변수를 초기화할 때 여러 계산을 수행해야 하는 경우 유용합니다.

```kotlin
val hexNumberRegex = run {
    val digits = "0-9"
    val hexDigits = "A-Fa-f"
    val sign = "+-"
    
    Regex("[$sign]?[$digits$hexDigits]+")
}

for (match in hexNumberRegex.findAll("+123 -FFFF !%*& 88 XYZ")) {
    println(match.value)
}
```