Kotlin은 코드 재사용을 지원하는 기능 중 하나인 'property delegation'를 통해, 공통된 프로퍼티 동작을 쉽게 추출할 수 있게 되었다.

이러한 예시 중 하나로 'lazy 프로퍼티'가 있으며, 이는 첫 번째로 접근할 때만 초기화되는 특성을 갖는 프로퍼티이다.  
이런 패턴은 코드 재사용 시 매우 유용하며, 이를 지원하지 않는 언어들은 필요할 때마다 수동으로 구현해야 한다.  
하지만, Kotlin에서는 'property delegation'을 통해 간편하게 구현할 수 있다.

표준 라이브러리에서는 'lazy 프로퍼티 패턴'을 구현하는 `lazy` 함수를 제공한다.

```kotlin
val value by lazy { createValue() }
```

반복되는 프로퍼티 패턴은 여러 가지가 있으며, 그 중 하나는 'Observable property'이다.  
'Observable property'는 값이 변경될 때마다 특정 동작을 실행하는 프로퍼티이다.

예를 들어, 프로퍼티의 변경 사항을 로그로 기록해야 하는 경우나, 리스트의 요소가 변경될 때마다 'Adapter'를 통해 화면을 새로 그리는 경우 등
표준 라이브러리는 이런 패턴을 쉽게 구현할 수 있도록 'Delegates.observable'을 제공한다.

```kotlin
var key: String? by Delegates.observable(null) { _, old, new ->
    println("Key changed from $old to $new")
}

var items: List<Item> by Delegates.observable(emptyList()) { _, _, _ ->
    notifyDataSetChanged()
}
```

'property delegation' 기능은 'lazy'와 'observable'를 포함하여, 언어 차원에서 다양한 프로그래밍 패턴을 추출 할 수 있게 해준다.  
좋은 예시로 'View and Resource Binding', 'Dependency Injection', 'DataBinding' 등이 있다.  

위 패턴 중 다수는 Java에서 어노테이션 처리가 필요하지만,
Kotlin에서는 'property delegation'을 통해 더 간결하고 'type-safe'하게 대체할 수 있다.

```kotlin
// View and resource binding
private val button: Button by bindView(R.id.button)
private val textSize by bindDimension(R.dimen.font_size)
private val doctor: Doctor by argExtra(DOCTOR_ARG)

// Dependency Injection using Koin
private val presenter: MainPresenter by inject()
private val repository: NetworkRepository by inject()
private val vm: MainViewModel by viewModel()

// Data binding
private val port by bindConfiguration("port")
pirvate val token: String by preferences.bind(TOKEN_KEY)
```

'property delegation'을 통해 '공통된 동작'을 어떻게 추출할 수 있는지 이해하기 위해,  
특정 프로퍼티의 'getter • setter'를 통해 변경 사항을 로그로 남기는 예시를 보자.

```kotlin
var token: String? = null
    get() {
        println("token return value : $field")
        return field
    }
    set(value) {
        println("token changed from $field to $value")
        field = value
    }

var failedLoginCount: Int = 0
    get() {
        println("failedLoginCount return value : $field")
        return field
    }
    set(value) {
        println("failedLoginCount changed from $field to $value")
        field = value
    }
```

위 예시의 두 프로퍼티는 타입이 다르지만 동작이 거의 유사하다.  
이러한 유사성은 프로젝트 내에서 자주 사용될 수 있는 패턴을 나타내며, 이를 'property delegation'을 통해 추출할 수 있다.

위임은 프로퍼티의 접근자, 즉 'val'의 경우 'getter'를, 'var'의 경우 'getter'와 'setter'를 기반으로 구현된다.  
이 접근자들은 다른 객체의 메서드로 위임 될 수 있으며, 'getter' → 'getValue' 함수로, 'setter' → 'setValue' 함수로 위임 된다.  
위임된 객체는 'by' 키워드 다음에 배치되며, 아래와 같이 생성할 수 있다.

```kotlin
var token: String? by LogginProperty(null)
var failedLoginCount: Int by LogginProperty(0)

private class LoggingProperty<T>(var value: T) {
    
    operator fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        println("${property.name} return value : $value")
        return value
    }

    operator fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: T
    ) {
        println("changed from ${property.name} to $value")
        this.value = value
    }
}
```

'property delegation'의 동작 원리를 완전히 이해하려면, 'by' 키워드가 실제로 컴파일될 때 발생하는 과정을 살펴보는게 좋다.  
예를 들어, 위 예시의 'token' 프로퍼티는 아래 코드처럼 컴파일된다.

```kotlin
@JvmField
private val `token$delegate` = LoggingProperty<Stirng?>(null)

var token: String?
    get() = `token$delegate`.getValue(this, ::token)
    set(value) { 
        `token$delegate`.setValue(this, ::token, value)  
    } 
```

'getValue'와 'setValue' 메서드는 단순히 값을 처리하는 것을 넘어서, 프로퍼티에 대한 구체적인 참조(::token)와 컨텍스트(this)를 함께 받는다.  
프로퍼티에 대한 참조는 대부분 프로퍼티의 이름을 얻는데 사용되며, 때로는 어노테이션에 대한 정보를 얻기위해 사용된다.  
컨텍스트는 함수가 어디서 사용되는지에 대한 정보를 제공한다.

다양한 컨텍스트 타입을 가진 'getValue'와 'setValue' 메서드가 여러 개 존재하면, 상황에 따라 적절한 메서드가 선택된다.  
개발자들은 이런 특성을 이용하여 아래 예시와 같이 다양한 뷰에서 사용되는 동일한 위임을 구현할 수 있다. 

```kotlin
class SwipeRefreshBinderDelegate(val id: Int) {
    private var cache: SwipeRefreshLayout? = null
    
    operator fun getValue(
        activity: Activity,     // Activity
        prop: KProperty<*>
    ): SwipeRefreshLayout {
        return cache ?: activity
            .findViewById<SwipeRefreshLayout>(id)
            .also { cache = it }
    }
    
    operator fun getValue(
        fragment: Fragment,     // Fragment
        prop: KProperty<*>
    ): SwipeRefreshLayout {
        return cache ?: fragment.view
            .findViewById<SwipeRefreshLayout>(id)
            .also { cache = it }
    }
} 
```