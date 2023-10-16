# Scope functions

## function selection

| 함수    | 객체 참조 방식 | 반환 값     | 확장 함수인지?               |
|:------|:---------|:---------|:-----------------------|
| let   | it       | 람다 블록 결과 | Yes                    |
| run   | this     | 람다 블록 결과 | Yes                    | 
| run   | -        | 람다 블록 결과 | No - 컨텍스트 객체 없이 사용한 경우 |
| with  | this     | 람다 블록 결과 | No - 컨텍스트 객체를 인자로 사용   |
| apply | this     | 객체 자신    | Yes                    |
| also  | it       | 객체 자신    | Yes                    |

### 일반적인 사용 가이드라인

- [let](#let) : `null`이 아닌 객체나 스코프 내에서 로컬 변수를 지정하여 사용할 때 사용합니다.
- [apply](#apply) : 객체를 설정할 때 사용합니다.
- [run](#run) : 객체를 구성하고 그 결과를 계산할 때 사용됩니다.
- [비확장 run](#run-비확장) : 표현식 대신 여러 문장을 실행할 때 사용합니다.
- [also](#also) : 객체를 그대로 반환하면서 추가적인 작업을 할 때 사용합니다.
- [with](#with) : 객체에서 여러 메서드를 연속으로 호출할 때 사용됩니다.

<img src="resources/Scope Function Guide.jpg" width="500"/>
---

### let

> - 객체 참조는 인자(`it`) 사용, 반환 값은 람다의 결과
> - 연속적인 연산의 결과를 변수에 할당하지 않고 처리하는 경우에 주로 사용
> - `?.` 연산자와 함께 사용하여 `null` 검사를 자동으로 수행할 때 사용
> - 스코프 내에서 새로운 로컬 변수를 사용할 때 사용

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

#### run 비확장
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

---

### apply

> - 객체 참조는 리시버(`this`) 사용, 반환 값은 객체 자신
> - 리시버 객체 속성을 설정할 때 주로 사용

`apply`는 컨텍스트 객체 자신을 반환하기에 객체 속성을 작업하는 코드 블록에 사용하는 것이 좋습니다.

즉, `apply`는 반환 값이 없고, 객체의 상태를 변경하는 코드에 사용됩니다. 그래서 반환 값을 사용할 필요가 없을 때 적합합니다.

```kotlin
val ys = Person("YongSuk").apply {
    age = 32
    city = "Korea"
}
println(ys)
```

---

### also

> - 객체 참조는 인자(`it`) 사용, 반환 값은 객체 자신
> - 객체의 속성이나 함수를 사용할 필요 없이 객체 자체에 대한 참조만 필요할 때 주로 사용
> - `also` 내부에서 외부 범위의 `this` 참조 시 유용하게 사용 가능

`also`는 컨텍스트 객체가 인자(`it`)로 제공되어 객체의 속성과 메서드보다는 객체 자체에 대한 참조가 필요한 작업을 할때 유용합니다.  
또한 `also`는 작업을 수행한 후에도 원래 객체를 반환하기에 호출 체인을 계속 유지할 수 있습니다.

```kotlin
val numbers = mutableListOf("one", "two", "three")
numbers
    .also { println("The list elements before adding new one: $it") }
    .add("four")
```

`also`는 외부 범위의 `this`를 참조하고 싶을 때에도 유용하게 사용할 수 있습니다.

```kotlin
class Outer {
    val name = "YongSuk"

    Inner().apply
    {
        println(this.name) // KimYongSuk
        println(this@Outer.name) // YongSuk
    }

    Inner().also
    {
        println(it.name) // KimYongSuk
        println(this.name) // YongSuk
    }

    inner class Inner {
        val name = "KimYongSuk"
    }
}
```

---

### takeIf and TakeUnless

> - 객체 참조는 인자(`it`) 사용, 반환 값은 객체 자신 또는 `null`
> - 단일 객체의 상태를 체크할 때 주로 사용하며, 스코프 함수에 연결하여 사용할 때 유용

`takeIf`와 `takUnless`는 단일 객체의 상태를 체크하는 로직을 함수 호출 체인에서 유용하게 사용됩니다.

객체와 함께 조건(`predicate`)를 호출하면, `takeIf`는 주어진 조건을 만족하면 해당 객체를 반환하고 그렇지 않으면 `null`을 반환합니다.  
`takeUnless`는 `takeIf`와 반대의 로직을 지니며 주어진 조건을 만족하면 `null`을 반환하고 그렇지 않으면 해당 객체를 반환합니다.

`takeIf`와 `takeUnless` 사용 시, 객체는 람다 인자(`it`)로 사용할 수 있습니다.

```kotlin
val number = Random.nextInt(100)

val evenOrNull = number.takeIf { it % 2 == 0 }
val oddOrNull = number.takeUnless { it % 2 == 0 }

println("even: $evenOrNull, odd: $oddOrNull")
```

`takeIf`와 `takeUnless`는 스코프 함수들(`let`, `also`, `run`, `apply` 등등)과 사용할 때 유용합니다.

`takeIf`와 `let`을 함께 사용하면 코드가 더 깔끔하고 가독성이 좋아집니다.  
아래 예제에서는 `takeIf`가 조건을 만족하는지 검사하고, 만족할 경우 `let` 블록을 실행합니다.  
이 방식을 사용하면 조건에 일치하는 경우에만 특정 로직을 실행할 수 있으며, 코드가 더 간결해집니다.

```kotlin
fun displaySubstringPosition(input: String, sub: String) {
    input.indexOf(sub).takeIf { it >= 0 }?.let {
        println("The substring $sub is found in $input.")
        println("Its start position is $it.")
    }
}

displaySubstringPosition("010000011", "11")
displaySubstringPosition("010000011", "12")
```

비교를 위해 아래는 `takeIf`나 스코프 함수를 사용하지 않고 같은 기능을 하는 함수 구현입니다.

```kotlin
fun displaySubstringPosition(input: String, sub: String) {
    val index = input.indexOf(sub)
    if (index >= 0) {
        println("The substring $sub is found in $input.")
        println("Its start position is $index.")
    }
}

displaySubstringPosition("010000011", "11")
displaySubstringPosition("010000011", "12")
```