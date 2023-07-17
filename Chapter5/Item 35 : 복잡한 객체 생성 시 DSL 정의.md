# Item 35: 복잡한 객체 생성 시 DSL 정의 고려

DSL이란 '도메인 특화 언어'의 줄임말로, Kotlin의 여러 특징들을 활용해 구성 가능합니다. 
이는 복잡한 객체나 객체의 계층 구조를 정의할 때 유용하며, 일단 정의되면 복잡성을 감추고 개발자가 의도를 명확하게 표현할 수 있게 해줍니다.

Kotlin DSL은 HTML을 표현하는데 좋은 방법 입니다.

```kotlin
body {
    div {
        a("https://kotlinlang.org") {
            target = ATarget.blank
            +"Main site"
        }
    }
    +"Some Content"
}
```

다른 플랫폼에서도 DSL을 사용하며 정의할 수 있으며, 아래는 Anko 라이브러리를 사용하여 정의된 Android View 입니다.

```kotlin
verticalLayout {
    val name = editText()
    button("Say Hello") {
        onClick { toast("Hello, ${name.text}!") }
    }
}
```

DSL은 데이터나 설정을 정의하는데 사용할 수 있습니다. 아래는 Ktor에서 API 정의한 예제 입니다.

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

Gradle에서도 DSL을 사용할 수 있습니다.

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

DSL을 사용하면 복잡하고 계층적인 데이터 구조를 쉽게 만들 수 있으며, DSL 내부에서는 Kotlin이 제공하는 모든 기능을 활용할 수 있습니다.
또한 Kotlin DSL은 완전한 타입 안전성을 보장하므로 개발 중 유용한 힌트를 얻을 수 있습니다.

---

## DSL 정의

DSL을 만드는 데 있어 함수 타입을 이해하는 것은 중요한 개념입니다.
함수 타입은 함수로 사용될 수 있는 객체를 표현하는 타입입니다.

예를 들어, `filter()`에서 원소를 추가 결정 조건(predicate)을 표현하는 데 함수 타입이 사용됩니다.

```kotlin
inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    val list= arrayListOf<T>()
    for (element in this) {
        if (predicate(element)) {
            list.add(element)
        }
    }
    return list
}
```

아래는 몇가지 함수 타입의 예시 입니다.

| 예시 | 설명 |
| --- | --- |
| `() -> Unit` | 인자를 받지 않고 `Unit`을 반환하는 함수 |
| `(Int) -> Unit` | `Int`를 인자로 받고 `Unit`을 반환하는 함수 |
| `(Int) -> Int` | `Int`를 인자로 받고 `Int`를 반환하는 함수 |
| `(Int, Int) -> Int` | `Int` 두개를 인자로 받고 `Int`를 반환하는 함수 |
| `(Int) -> () -> Unit` | `Int`를 인자로 받고 `Unit`을 반환하는 함수를 반환하는 함수 |
| `(() -> Unit) -> Unit` | `Unit`을 반환하는 함수를 인자로 받고 `Unit`을 반환하는 함수 |

함수 타입의 인스턴스를 생성하는 기본적인 방법은 다음과 같습니다.
- 람다 표현식 사용 (Using lambda expressions)
- 익명 함수 사용 (Using anonymous functions)
- 함수 참조 사용 (Using function references)

```kotlin
fun plus(a: Int, b: Int) = a + b

// 위 함수와 비슷한 함수

val plus1: (Int, Int) -> Int = {a, b -> a + b} // 람다 표현
val plus2: (Int, Int) -> Int = fun(a, b) = a + b // 익명 함수
val plus3: (Int, Int) -> Int = ::plus // 함수 참조
```

위 예제에서 프로퍼티 타입이 지정되어 있으므로 람다 표현식과 익명 함수의 인자 타입을 추론할 수 있습니다.

반대로 인자 타입을 지정하면 함수 타입을 추론할 수 있습니다.

```kotlin
val plus4 = { a: Int, b: Int -> a + b }
val plus5 = fun(a: Int, b: Int) = a + b
```

- 함수 타입 : 함수를 표현하는 객체를 나타내는데 사용됩니다.
- 익명 함수 : 일반 함수와 비슷해 보이지만 이름이 없습니다.
- 람다 표현식 : 익명 함수의 축약된 표기법 입니다.

함수 타입이 함수를 표현하게 해주면, 확장 함수에 대해서 다음과 같이 표현됩니다.

```kotlin
fun Int.myPlus(other: Int) = this + other
```

일반 함수를 이름 없이 정의하는 것처럼, 익명 확장 함수도 다음과 같이 정의할 수 있습니다.

```kotlin
val myPlus = fun Int.(other: Int) = this + other
```

아래는 수신 객체를 가진 함수 타입(function type with receiver)으로 일반 함수 타입과 비슷해 보이지만, 인자 앞에 수신 타입을 추가로 지정하고 Dot(.)으로 구분합니다.

```kotlin
val myPlus: Int.(Int) -> Int = fun Int.(other: Int) = this + other
```

이런 함수는 람다 표현식, 특히 수신 객체를 가진 람다 표현식을 사용하여 정의할 수 있습니다.
범위 내에서는 `this` 키워드가 확장 수신 객체를 참조합니다.

```kotlin
val myPlus: Int.(Int) -> Int = { other -> this + other }
```

익명 확장 함수 또는 수신 객체를 가진 람다 표현식을 사용하여 생성된 객체는 다음 3가지 방법으로 호출 할 수 있습니다.
- 일반 객체처럼 `invoke` 메서드 사용 : `myPlus.invoke(1, 2)`
- 확장 함수가 아닌 일반 함수처럼 사용 : `myPlus(1, 2)`
- 일반 확장 함수와 같이 사용 : `1.myPlus(2)`

수신 객체를 가진 함수 타입의 가장 중요한 특징은 `this`가 참조하는 것을 바꾼다는 것입니다.
이는 `apply()`에서 `this`를 사용하여 수신 객체의 메서드와 프로퍼티를 더 쉽게 참조하게 하는데 사용됩니다.

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