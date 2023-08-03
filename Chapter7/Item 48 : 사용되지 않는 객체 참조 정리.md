# Item 48: 사용되지 않는 객체 참조 정리

자동으로 메모리 관리를 해주는 언어에 익숙한 개발자들은 객체를 메모리에서 제거하는 것에 대해 잘 생각하지 않습니다.

예를 들어 Java에서는 Garbage Collector(GC)가 위 메모리 관리를 모두 수행합니다.
그러나 메모리 관리를 완전히 생각하지 않게되면 메모리 누수 및 일부 경우 OOM이 발생할 수 있습니다.

가장 중요한것은 더 이상 사용하지 않은 객체에 대한 참조를 유지해서는 안되고, 해당 객체가 메모리 측면에서 크거나 인스턴스가 많은 경우에 더욱 그렇습니다.

Android 주니어 개발자는 `Activity`에 대한 참조를 자유롭게 얻기 위해서 그것을 정적 또는 `companion object`의 속성으로 저장하는 실수를 할 수 있습니다.

```kotlin
class MainActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...
        activity = this
    }

    companion object {

        // 메모리 누수 발생
        var activity: MainActivity? = null
    }
}
```

`companion object`에서 `Activity` 참조를 보유하게 될 경우 Garbage Collector가 그것을 해제할 수 없습니다.
`Activity` 자체는 무거운 객체이므로 엄청난 메모리 누수가 발생될 수 있습니다.

이처럼 무거운 객체의 경우 정적으로 저장하는 대신 종속성을 관리하는 것이 좋습니다.
또한 다른 어떠한 것에 대한 참조를 저장하는 객체 보유 시 메모리 누수가 발생될 수 있음을 알아야 합니다.

```kotlin
class MainActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...

        //  전체 클래스에 대한 참조 누수
        logError = { Log.e(this::class.simpleName, it.message) }
    }

    companion object {

        // 메모리 누수 발생
        var logError: ((Throwable) -> Unit)? = null
    }
}
```

아래는 `Stack`의 구현의 예시로 다음과 같이 메모리 문제는 더 발견하기 어려울 수 있습니다.

```kotlin
class Stack {
    private var elements: Array<Any?> = arrayOfNulls(DEFAULT_INITIAL_CAPACITY)
    private var size = 0

    fun push(e: Any?) {
        ensureCapacity()
        elements[size++] = e
    }

    fun pop(): Any? {
        if (size == 0) throw EmptyStackException()
        return elements[--size]
    }

    private fun ensureCapacity() {
        if (elements.size == size) elements = elements.copyOf(2 * size + 1)
    }

    companion object {
        private const val DEFAULT_INITIAL_CAPACITY = 16
    }
}
```

위 코드에서 `pop()`실행 시 크기만 감소 시키고 배열의 요소를 해제하지 않는것을 문제점으로 볼 수 있습니다.

예를 들어 `Stack`에 1000개의 요소가 있고, 모두 하나씩 연달아 `pop()`하고 크기가 이제 1과 동일하다고 가정해보면,
이 `Stack`은 오직 하나의 요소만을 접근할 수 있고 가져야 하는 상황이 생깁니다.

그럼에도 불구하고 `Stack`은 여전히 1000개의 요소를 보유하고 있으며, GC는 이 요소들을 정리하지 못합니다.
이러한 요소들은 메모리를 낭비하고 있으며 이를 메모리 누수라고 합니다. 이 누수가 누적되면 OOM이 발생될 수 있습니다.

이를 해결하기 위해선 더 이상 필요하지 않을 시 배열에 `null`을 설정하면 됩니다.

```kotlin
fun pop(): Any? {
    if (size == 0) throw EmptyStackException()
    val result = elements[--size]
    elements[size] = null
    return result
}
```

위 `Stack`의 예시는 드물게 메모리 비용이 많이 드는 실수일 수 있지만, 위 규칙을 이용하여 이익을 얻을 수 있는 일상적인 객체가 있습니다.

예를 들어 `mutableLazy`라는 Property Delegate가 필요하다고 가정해보면 이는 `lazy`와 같이 동작해야 하지만, 프로퍼티 상태 변화를 허용해야 합니다.
아래 코드는 이를 구현한 것 입니다.

```kotlin
import kotlin.properties.ReadWriteProperty

fun <T> mutableLazy(initializer: () -> T): ReadWriteProperty<Any?, T> = MutableLazy(initializer)

private class MutableLazy<T>(
    val initializer: () -> T
) : ReadWriteProperty<Any?, T> {

    private var value: T? = null
    private var initialized = false

    override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        synchronized(this) {
            if (!initialized) {
                value = initializer()
                initialized = true
            }
            return value as T
        }
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        synchronized(this) {
            this.value = value
            initialized = true
        }
    }
}

var game: Game? by mutableLazy { readGameFromSave() }

fun setUpActions() {
    startNewGameButton.setOnClickListener {
        game = makeNewGame()
        startGame()
    }

    resumeGameButton.setOnClickListener {
        startGame()
    }
}
```

위 `mutableLazy` 구현은 `initializer`가 사용 후에 정리되지 않는 결점이 있습니다.
이는 더 이상 사용되지 않더라도 `MutableLazy`의 인스턴스에 대한 참조가 있는 한 `initializer`가 유지된다는 의미입니다.

아래는 `MutableLazy` 개선된 구현 방법 입니다.

```kotlin
import kotlin.properties.ReadWriteProperty

fun <T> mutableLazy(initializer: () -> T): ReadWriteProperty<Any?, T> = MutableLazy(initializer)

private class MutableLazy<T>(
    val initializer: (() -> T)?
) : ReadWriteProperty<Any?, T> {

    private var value: T? = null

    override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        synchronized(this) {
            val initializer = initializer
            if (initializer != null) {
                value = initializer()
                initialized = true
            }
            return value as T
        }
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        synchronized(this) {
            this.value = value
            this.initializer = null
        }
    }
}
```

위처럼 `initializer`를 `null`로 설정하면 이전 값은 GC에 의해 제거될 수 있습니다.

위와 같은 최적화 작업은 드물게 사용되는 객체에 대해서는 크게 신경쓰지 않아도 될 정도이지만, 미리미리 최적화하는 것은 나중에 사이드 이펙트를 적게 만드는 방법이 될 수 있습니다.
예를 들어 위에서 예시를 든 `Stack`은 무거운 객체를 저장하는데 사용될 수 있기에 최적화에 더욱 신경을 써야 합니다.

다음과 같이 많은 비용이 들지 않을 때 사용되지 않는 객체를 `null`로 설정하는 것은 좋습니다.

1. 많은 변수를 캡처할 수 있는 함수 타입
2. `Any`와 같이 알 수 없는 객체나 제네릭 타입인 경우

또한 라이브러리를 만들 때 위 최적화를 더욱 신경써야 합니다.

아래와 같이 Kotlin 표준 라이브러리의 모든 `lazy` Delegate 구현은 `initializer`가 사용 후 `null`로 설정되는 것을 볼 수 있습니다.

```kotlin
private class SynchronizedLAzyImpl<out T>(
    initializer: () -> T,
    lock: Any? = null
) : Lazy<T>, Serializable {

    private var initializer: (() -> T)? = initializer
    private var _value: Any? = UNINITIALIZED_VALUE
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value

                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)
}
```

프로그래밍에서 상태 유지 시 메모리 관리르 신중하게 고려해야 합니다.

메모리를 효율적으로 관리하지 않는 경우 느려진 성능, 프로그램 충돌 및 안정성 감소와 같은 문제가 발생될 수 있습니다.
따라서 구현을 변경하기 전, 프로젝트의 가장 효과적인 방안을 찾아야 하며, 이 방안은 메모리와 성능 뿐만이 아니라 코드의 가독성과 확장성도 고려해야 됩니다.

읽기 쉬운 코드는 일반적으로 잘 문서화되어 있고 이해하기 쉽기에 유지보수가 용이합니다. 그러나 성능과 메모리 측면에서 항상 최적이라고 할 수 없기에 확인이 필요합니다.
또한 라이브러리 개발 시 사용자가 어떻게 사용할지 알 수 없으므로 메모리 최적화와 성능이 중요합니다.

메모리 누수에 대한 경우는 다음과 같을 수 있습니다.

캐시는 사용되지 않을 수 있는 객체를 유지합니다. 이는 캐시의 주 목적이지만, 메모리 부족으로 인한 오류가 발생될 때는 크게 도움이 되지 않습니다.
이때는 캐시를 사용하는 것이 아닌, 약한 참조(weak reference)를 이용하여 메모리가 필요한 경우 CG에 의해 정리되어 메모리 부족으로 인한 오류를 방지할 수 있습니다.

일부 객체는 약한 참조를 사용하여 참조될 수 있으며 대표적으로 화면의 `Dialog`가 있습니다.
`Dialog`는 표시되는 한 CG에 정리되지 않으며 사용자의 의해 사라지게 되면 객체를 참조할 필요가 없기에 약한 참조를 사용하여 참조되는 대표적인 객체입니다.

그러나 메모리 누수를 예측하기 어렵고 앱이 충돌할 때까지 나타나지 않는 경우에 큰 문제가 될 수 있습니다.
특히 Android 앱의 경우, Window, Web 등에 사용되는 다른 클라이언트 어플리케이션 보다 메모리 사용에 대한 제한이 더 심각합니다.

위와 같은 이유로 인해 Android 플랫폼의 경우 메모리 누수를 찾기 위한 도구를 사용해야 합니다.  
가장 기본적인 도구는 Heap Profiler가 있으며 메모리 누수가 감지될 때마다 알림을 보내주는 LeakCanary 등이 있습니다.