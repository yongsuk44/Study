# Item 21: property delegate 사용하여 일반적인 Property 패턴 추출

Kotlin에서는 코드 재사용을 지원하는 Property Delegation 기능이 있습니다.
이 기능은 공통적으로 발생하는 Property 관련 동작을 쉽게 '재사용' 할 수 있습니다. 

`lazy` Property는 이러한 예시 중 하나로, 첫 번째로 접근할 때에만 초기화되는 특성을 가지고 있습니다. 
이러한 특성은 매우 유용하며, Property Delegation을 지원하지 않는 언어들은 필요할 때 마다 이를 수동으로 구현해야 하지만,
Kotlin에서는 Property Delegation을 이용해 이러한 동작을 쉽게 재사용할 수 있습니다.

```kotlin
val value by lazy { createValue() }
```

재사용 가능한 Property Pattern은 `lazy` 뿐만이 아니라 다른 것들도 존재합니다.
그 중 `observable` property는 값이 변경될 때마다 어떤 동작을 실행하는 property 입니다.

```kotlin
var key: String? by Delegates.observable(null) { _, old, new ->
    println("Key changed from $old to $new")
}
```

```kotlin
var items: List<Item> by Delegates.observable(emptyList()) { _, old, new ->
    redrawList(old, new)
}
```

`lazy`와 `observable` delegate는 언어 관점에서 특별한 것이 아닌, 더 일반적인 property delegation 메커니즘을 통해 추출될 수 있습니다.  
이 메커니즘은 다른 많은 패턴을 추출하는데 사용되며, 대표적으로 `View-Resource Binding`, `DI`, `Data Binding` 등이 있습니다.  
위 패턴들은 Java에서는 주로 주석 처리가 필요하지만, Kotlin에서는 쉽고 안전하게 property delegation을 통해 구현할 수 있습니다.

```kotlin
// View and resource binding
private val button: Button by bindView(R.id.button)
private val textSize by bindDimension(R.dimen.font_size)
private val doctor: Doctor by argExtra(DOCTOR_ARG)

//Dependency Injection using Koin
private val presenter: MainPresenter by inject()
private val repository: NetworkRepository by inject()
private val vm: MainViewModel by viewModel()

// Dat aBinding
private val port by bindConfiguration("port")
pirvate val token: String by preferences.bind(TOKEN_KEY)
```

위와 같이 다른 동작들을 Property Delegation을 사용해 추출할 수 있는지를 이해하려면, 
다음 예를 보면 쉽게 이해할 수 있습니다.

```kotlin
var token: String? = null
    get() {
        println("token return value : $field")
        return field
    }
    set(value) {
        println("Setting token: $value")
        field = value
    }

var failedLoginCount: Int = 0
    get() {
        println("failedLoginCount return value : $field")
        return field
    }
    set(value) {
        println("Setting failedLoginCount: $value")
        field = value
    }
```

위 property들은 타입은 다르지만, 동작이 거의 동일합니다.
이는 프로젝트에서 자주 사용할 수 있는 반복적인 패턴으로 보여지며, 이러한 동작들은 Property Delegation을 사용하여 추출 할 수 있습니다.

Delegation은 "Property가 접근자에 의해 정의된다" 라는 것을 기반으로 구현합니다.
- `val` → `getter`
- `var` → `getter`와 `setter`

위 메소드들은 다른 객체의 메소드로 위임될 수 있습니다.
 
- `getter` → `getValue` 함수로 위임
- `setter` → `setValue` 함수로 위임
 
그런 다음 그러한 객체를 `by` 키워드의 오른쪽에 배치합니다. 

위의 예제와 동일한 동작을 유지하려면, 다음과 같은 Delegate를 생성할 수 있습니다.

```kotlin
var token: String? by LogginProperty(null)
var failedLoginCount: Int by LogginProperty(0)

private class LoggingProperty<T>(var value: T) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        println("${property.name} return value : $value")
        return value
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        println("Setting ${property.name}: $value")
        this.value = value
    }
}
```

property delegation이 어떻게 동작하는지를 이해하려면, `by`가 어떻게 컴파일되는지를 확인 해보면 됩니다.  
위 `token` Property는 아래 코드와 비슷하게 컴파일됩니다.

```kotlin
@JvmField
private val `token$delegate` = LoggingProperty(null)

var token: String?
    get() = `token$delegate`.getValue(this, ::token)
    set(value) = `token$delegate`.setValue(this, ::token, value)
```

`getValue`와 `setValue` 메소드는 값뿐만 아닌, Property 자체에 대한 참조(`::token`)와 현재 인스턴스(`this`)도 받습니다.

여러 `getValue`와 `setValue` 메소드를 가지고 있지만 컨텍스트 유형이 다른 경우, 다른 상황에서 다른 것이 선택됩니다. 
이 사실은 창의적인 방식으로 사용될 수 있습니다.