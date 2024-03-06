Kotlin의 여러 기능을 함께 사용하면 'configuration'을 구성하는 것과 같이 DSL을 만들 수 있다.
이러한 DSL은 복잡한 객체나 객체의 계층 구조를 정의할 때 유용하다.
정의하기는 쉽지 않지만, 일단 정의되면 보일러 플레이트와 복잡성을 숨기고 개발자의 의도를 명확하게 표현할 수 있다.

예를 들어, Kotlin DSL은 HTML을 표현하는데 인기있는 방법이며, 클래식 HTML과 React HTML을 모두 표현할 수 있습니다.

```kotlin
body {
    div {
        a("https://kotlinlang.org") {
            target = ATarget.blank
            +"Main site"
        }
    }
    +"Some content"
}
```

다른 플랫폼의 'View'도 DSL을 사용하여 정의할 수 있다.
아래는 'Anko' 라이브러리를 통해 정의된 간단한 'Android View'이다.

```kotlin
verticalLayout {
    val name = editText()
    button("Say Hello") {
        onClick { toast("Hello, ${name.text}!") }
    }
}
```

Desktop 애플리케이션도 마찬가지로, JavaFX를 기반으로 구축된 TornadoFX에서 정의된 View이다.

```kotlin
class HelloWorld : View() {
    override val root = hbox {
        label("Hello, world!") {
            addClass(heading)
        }

        textfield {
            promptText = "Enter your name"
        }
    }
}
```

DSL은 데이터나 'configuration'을 정의할 때도 자주 사용되며, 다음은 Ktor의 API 정의이다.

```kotlin
fun Routing.api() {
    route("news") {
        get {
            val newsData = NewsUseCase.getAcceptedNewsData()
            call.respond(newsData)
        }

        get("propositions") {
            requireSecret()
            val newsData = newsUseCase.getPropositions()
            call.respond(newsData)
        }
    }
}
```

그리고 다음은 Kotlin Test에서 정의된 테스트 케이스 스펙들이다.

```kotlin
class MyTests : StringSpec({
    "length should return size of string" {
        "hello".length shouldBe 5
    }
    "startsWith should test for a prefix" {
        "world" should startWith("wor")
    }
})
```

Gradle DSL을 사용하여 Gradle configuration을 정의할 수도 있다.

```kotlin
plugins {
    `java-library`
}

dependencies {
    api("junit:junit:4.12")
    implementation("junit:junit:4.12")
    testImplementation("junit:junit:4.12")
}

configurations {
    implementation {
        resoulutionStrategy.failOnVersionConflict()
    }
}

sourceSets {
    main {
        java.srcDirs("src/core/java")
    }
}

java {
    sourceCompatibility = JavaVersion.VERSION_11
    targetCompatibility = JavaVersion.VERSION_11
}

tasks {
    test {
        testLogging.showExceptions = true
    }
}
```

DSL을 사용하면 복잡하고 계층적인 데이터 구조를 쉽게 만들 수 있다. 이러한 DSL 내에서는 Kotlin이 제공하는 모든 기능을 사용할 수 있으며, Kotlin DSL은 Groovy와 달리 완전히 '
type-safe'하다.
이미 어떤 Kotlin DSL을 사용해봤을 수도 있지만, 직접 정의하는 방법을 알아두는 것도 중요하다.

## Defining your own DSL

자신만의 DSL을 만드는 방법을 이해하기 위해서는 receiver가 있는 함수 타입의 개념을 이해하는 것이 중요하다.  
그 전에, 함수 타입 자체에 대한 개념을 간단히 살펴보자. 함수 타입은 함수로 사용될 수 있는 객체를 나타내는 타입이다.

예를 들어, 'filter'에서는 특정 요소가 허용될 수 있는지를 결정하는 'predicate'를 나타내는데 사용된다.

```kotlin
inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    val list = arrayListOf<T>()
    for (element in this) {
        if (predicate(element)) {
            list.add(element)
        }
    }
    return list
}
```

아래는 함수 타입의 몇가지 예시들이다.

| 예시                     | 설명                                                         |
|------------------------|------------------------------------------------------------|
| `() -> Unit`           | 인자가 없고, `Unit`을 반환하는 함수                                    |
| `(Int) -> Unit`        | 인자로 `Int`를 받고, `Unit`을 반환하는 함수                             |
| `(Int) -> Int`         | 인자로 `Int`를 받고, `Int`를 반환하는 함수                              |
| `(Int, Int) -> Int`    | `Int` 타입의 두 인자를 받고, `Int`를 반환하는 함수                         |
| `(Int) -> () -> Unit`  | 인자로 `Int`를 받고, 다른 함수를 반환하는 함수, 이 다른 함수는 인자가 없고 `Unit`을 반환  |
| `(() -> Unit) -> Unit` | 다른 함수를 인자로 받고, `Unit`을 반환하는 함수, 이 다른 함수는 인자가 없고 `Unit`을 반환 |

함수 타입의 인스턴스를 생성하는 기본적인 방법은 다음과 같다.

- 람다 표현식 사용 (Using lambda expressions)
- 익명 함수 사용 (Using anonymous functions)
- 함수 참조 사용 (Using function references)

예를 들어 다음 함수를 생각해보자.

```kotlin
fun plus(a: Int, b: Int): Int = a + b
```

이와 유사한 기능을 하는 함수를 다음과 같은 방법으로 생성할 수 있다.

```kotlin
val plus1: (Int, Int) -> Int = { a, b -> a + b }    // lambda expression
val plus2: (Int, Int) -> Int = fun(a, b) = a + b    // anonymous function
val plus3: (Int, Int) -> Int = ::plus               // function reference
```

위 예시에서는 프로퍼티의 타입이 지정되어 있기 때문에 람다 표현식과 익명 함수 내의 인자 타입을 추론할 수 있으며,
이와 반대로 인자 타입을 명시하면 함수 타입을 추론할 수 있다.

```kotlin
val plus4 = { a: Int, b: Int -> a + b }
val plus5 = fun(a: Int, b: Int) = a + b
```

함수 타입은 함수를 나타내는 객체를 나타내는데 사용된다.  
익명 함수는 일반 함수와 비슷해 보이지만 이름이 없다.  
람다 표현식은 익명 함수를 위한 더 짧은 표기법이다.

함수 타입을 가지고 함수를 나타낼 수 있다면, 다음과 같은 확장 함수도 표현할 수 있을 것이다.

```kotlin
fun Int.myPlus(other: Int) = this + other
```

앞에서 익명 함수는 일반 함수와 동일한 방식으로 선언하여 생성할 수 있다고 설명하였다.  
익명 확장 함수도 같은 방식으로 정의할 수 있다.

```kotlin
val myPlus = fun Int.(other: Int) = this + other
```

위 예시의 확장 함수는 어떤 타입을 가지고 있을끼? 확장 함수를 나타내기 위한 특별한 타입이 있다. 이를 '리시버를 가진 함수 타입'이라고 한다.
리시버를 가진 함수 타입은 일반 함수 타입과 유사해 보이지만, 인자들 앞에 리시버 타입을 추가로 명시하고, 이들을 Dot(.)으로 구분한다.

```kotlin
val myPlus: Int.(Int) -> Int = fun Int.(other: Int) = this + other
```

위와 같은 함수는 '리시버를 가진 람다 표현식'을 사용하여 정의할 수 있다.  
이는 람다 표현식 범위 내에서 'this' 키워드가 확장 리시버(위와 같은 경우에는 'Int' 타입의 인스턴스)를 참조하기 때문이다.

```kotlin
val myPlus: Int.(Int) -> Int = { other -> this + other }
```

'리시버를 가진 익명 확장 함수 또는 '리시버를 가진 람다 표현식'으로 생성된 객체는 다음 3가지 방법으로 호출 할 수 있습니다.

```kotlin
myPlus.invoke(1, 2)     // 표준 객체처럼 'invoke' 메서드를 사용
myPlus(1, 2)            // 확장 함수가 아닌 일반 함수처럼 사용
1.myPlus(2)             // 일반 확장 함수와 같이 사용
```

'리시버를 가진 함수 타입'의 가장 중요한 특징은 'this'가 가리키는 대상이 바뀐다는 것이다.  
예를 들어, 'apply' 함수에서는 리시버 객체의 메서드와 프로퍼티를 참조하기 쉽게 만들기 위해 'this'를 사용한다.

```kotlin
inline fun <T> T.apply(block: T.() -> Unit): T {
    this.block()
    return this
}

class User {
    var name: String = ""
    var surname: String = ""
}

val user = User().apply {
    name = "John"
    surname = "Doe"
}
```

리시버를 가진 함수 타입은 Kotlin DSL의 가장 기본적인 구성 요소이다.
이를 활용하여, HTML Table을 생성하는 간단한 DSL을 만들어보자.

```kotlin
fun createTable(): TableDsl = table {
    tr {
        for (i in 1..2) {
            td {
                +"This is column $i"
            }
        }
    }
}
```

위 DSL의 시작부에서 'table' 함수를 볼 수 있다. 어떤 리시버도 없는 Top-level에 있기에, 이는 Top-level 함수여야 한다.
그러나, 'table' 함수 인자 내부에서 'tr' 함수를 사용하는 것을 볼 수 있다. 'tr' 함수는 오직 'table' 정의 내에서만 사용할 수 있어야 한다.
이것이 'table' 함수 인자가 이런 함수를 가진 리시버를 가져야 하는 이유이다.
비슷하게 'tr' 함수 인자 역시 'td' 함수를 포함하는 리시버를 가져야 한다.

```kotlin
fun table(init: TableBuilder.() -> Unit): TableBuilder { /* .. */ }

class TableBuilder {
    fun tr(init: TrBuilder.() -> Unit) { /* .. */ }
}

class TrBuilder {
    fun td(init: TdBuilder.() -> Unit) { /* .. */ }
}

class TdBuilder
```

`+"This is row $i"`는 문자열에 대한 'unary plus' 연산자이다. 이는 'TdBuilder'에 정의되어 있어야 한다.

```kotlin
class TdBuilder {
    var text = ""

    operator fun String.unaryPlus() {
        text += this
    }
}
```

이제 DSL이 잘 정의되었다. 이를 제대로 작동시키기 위해서, 모든 단계에서 'builder'를 생성하고, 함수 파라미터(init)를 사용하여 초기화해야 한다.
이 과정을 통해 'builder' 인스턴스는 'init' 함수에서 지정된 모든 데이터를 포함하게 된다.
최종적으로 이 'builder'를 반환하거나, 이 데이터를 포함하는 다른 객체를 생성할 수 있지만 이 예제에서는 그저 'builder'를 반환할 것이다.

'table' 함수는 아래와 같이 정의할 수 있다.

```kotlin
fun table(init: TableBuilder.() -> Unit): TableBuilder {
    val tableBuilder = TableBuilder()
    init.invoke(tableBuilder)
    return tableBuilder
}
```

'table' 함수를 더 간결하게 만들기 위해 'apply' 함수를 사용할 수 있다.

```kotlin
fun table(init: TableBuilder.() -> Unit) =
    TableBuilder().apply(init)
```

마찬가지로, DSL의 다른 부분에서도 더 간결하게 만들기 위해 'apply' 함수를 사용할 수 있다.

```kotlin
class TableBuilder {
    var trs = listOf<TrBuilder>()

    fun tr(init: TrBuilder.() -> Unit) {
        trs = trs + TrBuilder().apply(init)
    }
}

class TrBuilder {
    var tds = listOf<TdBuilder>()

    fun td(init: TdBuilder.() -> Unit) {
        tds = tds + TdBuilder().apply(init)
    }
}
```

## When should we use it?

DSL은 정보를 정의하는 방법을 제공한다. 원하는 어떤 종류의 정보든 표현할 수 있지만, 사용자에게 이 정보가 나중에 어떻게 사용될지 명확하지 않다.
Anko, TornadoFX, HTML DSL에서는 정의에 기반하여 View가 올바르게 구축될 것이라고 믿지만, 정확히 어떻게 구축되는지 추적하기 어려울 수 있다.
좀 더 복잡한 사용 사례는 발견하기 어려울 수 있고, DSL 선언 방식에 익숙하지 않은 개발자들에게는 사용하기 혼란스러울 수 있다.
유지보수는 말할 것도 없다. DSL이 정의되는 방식은 개발자의 혼란과 성능 측면에 있어 비용이 될 수 있다.
DSL은 다른 간단한 기능 대신 사용할 수 있는 경우에는 'overkill'이라고 볼 수 있다. 그러나, 다음 같은 경우에는 DSL이 유용할 수 있다.

- 복잡한 데이터 구조
- 계층 구조
- 방대한 양의 데이터 표현 시

DSL은 특정 구조에 대한 보일러플레이트를 없애는 것에 초점을 맞춘다.
'builder pattern'이나 'constructor'만을 사용하여 모든 것을 표현할 수 있지만, DSL은 이러한 구조의 보일러플레이트를 제거하는데 효과적이다.
Kotlin은 더 간단한 기능만으로는 해결할 수 없고, 반복적인 보일러플레이트 코드가 나타날 때 DSL 사용을 고려해야 한다.
