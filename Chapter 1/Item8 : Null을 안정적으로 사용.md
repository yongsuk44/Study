# Null 안정적으로 사용하기

Kotlin은 null 참조를 안전하고 관리하기 쉽게하는 많은 기능을 제공합니다. Kotlin의 안전 호출, 엘비스 연산자, 스마트 캐스팅 등의 기능을 활용하면, 자주 발생하는 Null Pointer 예외를 예방할 수 있습니다.

## Null이란?

`null`은 값이 없음을 표시하며, 속성의 경우 값이 설정되지 않았거나 삭제됨을 의미합니다. 함수가 `null`을 반환하는 경우는 각 함수에 따라 의미가 다르며, `null`이 반환되면 이를 처리해야하며, 이 처리 방식은 API 사용자가 결정합니다.

예를 들어, 문자열을 Int로 변환할 수 없을 때 또는 조건에 일치하는 요소가 없을 때 `null`을 반환할 수 있습니다.

```kotlin
fun String.toIntOrNull(): Int? 

Iterable<T>.firstOrNull(() -> Boolean)
```

## Null을 안전하게 사용하기

`null`을 안전하게 처리하는 가장 보편적인 방법은 안전한 호출(safe call)과 스마트 캐스팅(smart casting)을 사용하는 것입니다.

```kotlin
printer?.print() // 안전한 호출
if (printer != null) printer.print() // 스마트 캐스팅
```

이 두 방법 모두, `print()`는 `printer`가 `null`이 아닐 때만 호출됩니다. 이는 사용자와 개발자 모두에게 안전하며 편리한 옵션입니다.

### 안전한 호출

Kotlin에서는 `nullable` 변수를 처리하는 방법이 다양하며, 그 중 엘비스 연산자를 사용하여 `null` 타입에 대한 기본 값을 제공하는 것이 일반적입니다.

```kotlin
val printerName1 = printer?.name ?: "Unknown"
val printerName2 = printer?.name ?: throw IllegalStateException("Printer is not set")
val printerName3 = printer?.name ?: return
```

엘비스 연산자의 오른쪽에는 `return`이나 `throw`를 포함한 어떤 표현식도 사용할 수 있습니다. 또한, `null` 대신 비어있는 리스트를 요청하는 것이 일반적이므로 `Collection<T>?.orEmpty()` 확장 함수를 사용하여 `null`이 아닌 `List<T>`를 반환할 수 있습니다.

### 스마트 캐스팅

스마트 캐스팅은 Kotlin의 contracts 기능을 통해 지원됩니다. contracts 기능을 사용하면 함수 내에서 스마트 캐스팅을 할 수 있습니다.

```kotlin
println("Name?")
val name = readLine()
if (!name.isNullOrBlank()) println("Hello, $name")

val news: List<News>? = getNews()
if (!news.isNullOrEmpty()) news.forEach { notify(it) }
```

---

## Throw 에러 던지기
Kotlin에서 null을 안정적으로 사용하는 데 있어 중요한 요소 중 하나는 예외 처리입니다.
특히 Kotlin의 [안전한 호출](#안전한-호출)(`?.`)이 제공하는 편리함에도 불구하고, `null` 객체에 대한 메소드 호출이 조용히 무시되는 문제가 발생할 수 있습니다.
이로 인해 개발자가 프로그램의 상태를 정확하게 이해하기 어렵고, 발생되는 버그를 추적하고 해결하는데 어려움을 겪을 수 있습니다.

### Null Throw 처리
Kotlin에서는 아래와 같은 방법으로 문제를 해결하고 프로그램의 안정성을 높일 수 있습니다.

- **not-null 연산자 (`!!`)**: 이 연산자는 `null`이 아닌 객체에 대한 참조를 반환하거나, 객체가 `null`인 경우 `NullPointerException`을 발생시킵니다.
  ```kotlin
  val printer: Printer? = getPrinter()
  printer!!.print()  // printer가 null이면 NullPointerException 발생
  ```

- **`requireNotNull()`**: 이 함수는 주어진 인자가 `null`인지 확인하고, `null`이 아니면 인자를 반환합니다. 인자가 `null`인 경우, 이 함수는 `IllegalArgumentException`을 발생시킵니다.
  ```kotlin
  val printer: Printer? = getPrinter()
  requireNotNull(printer).print()  // printer가 null이면 IllegalArgumentException 발생
  ```

- **`checkNotNull()`**: 이 함수는 `requireNotNull()`와 비슷하지만, 인자가 `null`인 경우 `IllegalStateException`을 발생시킵니다.
  ```kotlin
  val printer: Printer? = getPrinter()
  checkNotNull(printer).print()  // printer가 null이면 IllegalStateException 발생
  ```

- **직접 예외 던지기**: 필요에 따라 직접 정의한 예외를 던질 수도 있습니다.
  ```kotlin
  val printer: Printer? = getPrinter()
  if (printer == null) throw PrinterNotFoundException()
  printer.print()
  ```

이렇게 null 처리에 대한 안전성과 예외 처리를 적절히 조합하여 사용하면, 프로그램의 안정성을 높이고 개발자가 프로그램 상태를 더 잘 이해할 수 있도록 도와줍니다.

---

## not-null `!!` 연산자 사용과 주의점
### `!!` 연산자 사용하기
Kotlin에서 `null` 가능성을 처리하는 가장 단순한 방법 중 하나는 not-null 연산자 `!!`를 사용하는 것입니다.   
이 연산자를 사용하면, 값이 `null`이 아님을 확신하고 이를 사용할 수 있습니다. 하지만, 이 확신이 틀릴 경우 `NullPointerException`이 발생합니다.  
이는 코드의 가독성을 저하시키며 디버깅을 어렵게 할 수 있습니다. 또한, 이 연산자의 간결함 때문에 남용되거나 잘못 사용될 가능성이 있습니다.

`!!` 연산자는 주로 유형이 `nullable`이지만 `null`이 예상되지 않는 상황에서 사용됩니다.   
그러나 이런 상황에서의 문제점은 현재 예상되지 않을지라도 향후에 언제든 `null`이 될 수 있다는 점입니다. 
이 연산자는 실질적으로 `null` 가능성을 감추며 이는 잠재적인 버그의 원인이 될 수 있습니다.

다음 예시를 살펴보겠습니다.
```kotlin
fun largestOf(a: Int, b: Int, c: Int, d: Int): Int =
    listOf(a, b, c, d).max()!!
```

이런 상황에서는, 우리는 `listOf(a, b, c, d)`가 비어 있을 수 없다는 것을 알고 있습니다. 
따라서 `max()` 함수는 `null`을 반환할 수 없습니다. 
그러나 만약 아래와 같이 vararg를 사용하여 함수를 호출한다면 상황이 달라집니다.

```kotlin
fun largestOf(vararg numbers: Int): Int =
    numbers.max()!!
```

이 경우, `numbers`는 비어 있을 수 있습니다. 따라서 `max()` 함수는 `null`을 반환할 수 있습니다. 이런 경우에는 `NullPointerException`이 발생할 수 있습니다.   

따라서, `!!` 연산자는 꼭 필요한 상황에서만 사용하고, 가능한 한 `null`에 대한 안전한 코드를 작성하는 것이 좋습니다.

### `null` 가능성 정보의 은폐
nullability에 대한 정보가 은폐되면 중요한 상황에서 이를 쉽게 놓칠 수 있습니다. 이는 변수에 대해서도 동일하게 적용됩니다.

```kotlin
class UserRepositoryTest {
    private var dao: UserDao? = null
    private var repository: UserRepository? = null
    
    @Before
    fun set() {
        dao = mockk()
        repository = UserRepository(dao!!)
    }
    
    @Test
    fun test() {
        repository!!.getUser()
    }
}
```

위의 코드에서 `dao`와 `repository` 속성을 매번 언팩하는 것은 귀찮습니다. 또한 이러한 속성이 향후에 의미 있는 `null`값을 가질 수 있는 가능성을 제한합니다.
따라서 변수를 `null`로 초기화하고 `!!` 연산자를 사용하는 것은 권장되지 않습니다. 

이런 상황에 대처하는 더 좋은 방법은 `lateinit` 또는 `Delegates.notNull`을 사용하는 것입니다. 
이 방법들을 사용하면 `null` 가능성을 명확하게 표시하면서 코드를 더 간결하게 작성할 수 있습니다.

---

## 불필요한 nullability 피하기
Nullability는 처리 비용이 발생할 수 있기에 올바르게 관리되어야 하므로 필요하지 않을 때는 nullability를 피하는 것이 좋습니다.  

`null`은 중요한 메시지를 전달할 수 있지만, 다른 개발자에게는 의미 없는 상황을 제공하게 되어 피해야 합니다. 그렇지 않으면,
그들은 안전하지 않은 not-null 연산자 `!!`를 사용하거나, 코드를 불필요하게 복잡하게 만드는 일반적인 `null` 안전 처리를 반복하게 될 수 있습니다.

이를 위한 방법은 다음과 같습니다

| 방법 | 설명 |
| --- | --- |
| **결과를 예상하는 함수의 변형 제공** | 클래스는 결과가 기대되는 함수의 변형을 제공하여, 값의 부재를 고려하여 nullable한 결과 또는 `sealed class <Result>`를 반환할 수 있습니다. `List<T>`의 `get` 및 `getOrNull`과 같은메소드가 이에 해당하는 간단한 예시입니다. |
| **lateinit 프로퍼티 또는 notNull 대리자 사용** | 클래스 생성보다는 늦게, 하지만 사용하기 전에 확실히 값이 설정되는 경우, `lateinit` 프로퍼티나 `notNull` 대리자를 사용할 수 있습니다. |
| **null 대신 빈 컬렉션 반환** | `List<Int>?` 또는 `Set<String>?`과 같은 컬렉션을 반환할 때, `null`이 아닌 빈 컬렉션을 반환합니다. `null`은 컬렉션이 없음을 의미합니다. 요소의 부재를 나타내기 위해서는 빈 컬렉션을 사용하세요. |
| **nullable enum과 none enum 값 구분** | `null`과 `none`은 두 가지 다른 메시지입니다. `null`은 별도로 처리되어야 하는 특수한 메시지입니다. enum 정의에는 포함되지 않으므로 원하는 enum에 추가할 수 있습니다. |

---

## lateinit 프로퍼티와 notNull 대리자
프로젝트에서 클래스 생성 중에 초기화 할 수 없지만 첫 번째 사용 전에 확실히 초기화될 필요가 있는 프로퍼티를 가지는 경우는 흔한 일입니다.   
이 문제에 대한 올바른 해결책은 lateinit 수정자를 사용하여 나중에 프로퍼티를 초기화하는 것입니다

```kotlin
class UserRepositoryTest {
    private lateinit var dao: UserDao
    private lateinit var repository: UserRepository
    
    @Before
    fun set() {
        dao = mockk()
        repository = UserRepository(dao)
    }
    
    @Test
    fun test() {
        repository.getUser()
    }
}
```
`lateinit` 변수의 비용은 변수가 초기화되기 전에 값을 가져오려고 시도하면 예외가 발생한다는 것입니다. 이는 예외에 대한 걱정으로 번질 수 있지만, 개발자들이 원하는 동작입니다.

`lateinit`을 사용할 때는 변수가 첫 번째 사용 전에 반드시 초기화될 것이라는 것을 알고 있어야 합니다. 만약 이를 잘못 판단했다면, 이에 대해 알고 수정 할 수 있어야 합니다. 

`lateinit`과 nullable 사이의 주요 차이점은 다음과 같습니다:

- nullable에서 not-null로 "언팩"하는 작업이 필요하지 않습니다.
- 나중에 null을 사용해 의미를 전달해야 하는 경우, 쉽게 nullable로 전환할 수 있습니다.
- 프로퍼티가 초기화된 후에는 초기화되지 않은 상태로 되돌릴 수 없습니다.

`lateinit`은 변수가 첫 번째 사용 전에 반드시 초기화될 것이라는 것이 확실한 경우에 이상적입니다.   
클래스가 생명주기를 가지고 있으며, 이 생명주기 중 어느 시점에서 프로퍼티를 설정할 목적으로 첫 번째 호출되는 메소드가 있는 경우 이런 상황에 해당합니다.
예를 들어, Android에서 `Activity`의 `onCreate()`메소드에서 객체를 설정하는 상황이 이에 해당합니다.

그러나 `lateinit`은 모든 상황에 적용할 수 있는 것이 아닙니다. 
예를 들어, JVM에서 기본형에 해당하는 타입(Int, Long, Double, Boolean 등)으로 프로퍼티를 초기화해야 하는 경우에는 `lateinit`을 사용할 수 없습니다.  
이런 경우에는 `Delegates.notNull`을 사용해야 합니다. `Delegates.notNull`은 약간의 성능 저하가 있지만,
Kotlin에서는 해당 타입을 지원하므로 안전하게 사용할 수 있습니다.

```kotlin
class UserActivity: Activity() {
    private var userAge: Int by Delegates.notNull()
    private var isNotification: Boolean by Delegates.notNull()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        userAge = intent.getIntExtra("userAge", 0)
        isNotification = intent.getBooleanExtra("isNotification", false)
    }
}
```
이러한 경우에는 종종 프로퍼티 대리자(property delegate)로 대체 될 수 있습니다.  
위의 예제에서는 `onCreate()`에서 인자를 읽는 대신, 해당 프로퍼티를 lazy하게 초기화하는 대리자를 사용할 수 있습니다.

```kotlin
class UserActivity: Activity() {
    private var userAge: Int by lazy { intent.getIntExtra("userAge", 0) }
    private var isNotification: Boolean by lazy { intent.getBooleanExtra("isNotification", false) }
}
```