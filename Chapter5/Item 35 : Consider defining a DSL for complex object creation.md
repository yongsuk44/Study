# Item 35: Consider defining a DSL for complex object creation

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
