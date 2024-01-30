`null`은 값이 없음을 의미하며, 프로퍼티에서는 값이 설정되지 않았거나 제거되었음을 의미한다.  
함수가 `null`을 반환할 때는 함수에 따라 다른 의미를 가질 수 있다.

- `String.toIntOrNull()`은 문자열이 유효한 정수 타입이 아니면 `null` 반환
- `Iterable<T>.firstOrNull(() -> Boolean)` 주어진 조건에 맞는 첫 번째 요소를 반환하며, 조건에 맞는 요소가 없으면 `null` 반환

이와 같은 상황뿐만 아니라, 다른 모든 상황에서도 `null`의 의미는 명확해야 한다.  
'Nullable' 값들은 반드시 처리되어야 하며, 이를 어떻게 처리할지 결정하는 것은 해당 함수를 사용하는 개발자의 몫이다.

```kotlin
val printer: Printer? = getPrinter()
printer.print()                         // Compilation Error : printer is nullable and cannot be used directly

printer?.print()                        // Safe-call, used only if printer is not null
if (printer != null) printer.print()    // Smart-casting
printer!!.print()                       // Not-null assertion, throws NPE when printer is null
```

일반적으로 'Nullable 타입'을 처리하는 방법은 다음과 같이 있다.

- 'Safe-call(`?.`)', 'Smart-casting', 'Elvis-연산(`?:`)' 등을 사용하여 'Nullability'를 안전하게 처리
- Throw an error
- 해당 함수나 프로퍼티를 리팩토링하여 'Non-nullable 타입'으로 변경

## Handling nulls safely

위에서 언급한 것과 같이 `null`을 안전하게 처리하는 일반적인 방법은 'Safe-call'과 'Smart-casting'을 사용하는 것이다.

```kotlin
printer?.print()                        // Safe-call
if (printer != null) printer.print()    // Smart-casting
```

두 방법 모두, `print()`는 `printer`가 `null`이 아닐 때에만 호출되며, 이는 개발자 입장에서 가장 안전하고 편리한 옵션이다.

또 하나의 일반적인 방법은 'Elvis 연산'을 사용하여 'Nullable 타입'에 대한 기본 값을 제공하는 것이다.  
'Elvis 연산'은 오른쪽 항에 `return`과 `throw`를 포함한 어떠한 표현식도 사용할 수 있다.

```kotlin
val printerName1 = printer?.name ?: "Unknown"
val printerName2 = printer?.name ?: throw IllegalStateException("Printer is not set")
val printerName3 = printer?.name ?: return
```

또한 많은 객체들은 추가적으로 Kotlin에서 제공하는 확장 함수를 통해 `null`에 대한 기본 값을 제공한다.  
예를 들어, `null` 대신 비어있는 컬렉션을 제공하는 `Collection<T>?.orEmpty()`를 통해 `List<T>`를 반환하여,
'Nullable 타입'을 'Non-nullable 타입'으로 변환할 수 있다.

'Smart-casting'은 함수 내에서 'contracts' 기능을 통해 지원된다.

```kotlin
println("What is your name?")
val name = readLine()
if (!name.isNullOrBlank()) {
    println("Hello, ${name.uppercase()}")
}

val news: List<News>? = getNews()
if (!news.isNullOrEmpty()) {
    news.forEach { notifyUser(it) }
}
```

---

## Throw an error

'Safe-call'의 한 가지 문제는 `printer`가 절대 `null` 아니라고 판단하고, `print()`를 호출하려고 시도할 때 생긴다.
만약, 이때 `printer`가 `null`일 경우에 `print()`가 호출되지 않는 상황이 발생할 것이다. 이와 같은 시도는 오류의 근본 원인을 숨기고 문제 해결을 어렵게 만든다.
이처럼 어떤 것에 대한 기대에 반하는 상황이 발생했을 때, 개발자에게 알리기 위해 오류를 발생시키는 것이 더 좋다.
오류를 발생시킬 떄에는 `throw`를 사용하거나 'Not-null assertion(`!!`)', `requireNotNull()`, `checkNotNull()` 등을 사용하여 수행 할 수 있다.

```kotlin
fun process(user: User) {
    requireNotNull(user.name)
    val context = checkNotNull(context)
    val networkService = getNetowkrService(context) ?: throw NoInternetConnection()

    networkService.getData { data, userData ->
        show(data!!, userData!!)
    }
}
```

---

## The problems with the not-null assertion `!!`

'Nullable 타입'을 처리하는 가장 간단한 방법은 'Not-null assertion(`!!`)'을 사용하는 것이다.  
이는 개념적으로 Java에서 일어나는 일과 유사하다. - 어떤 값이 'null'이 아니라고 생각하고, 그것이 틀렸을 경우 'NPE'가 발생 된다. -  

'Not-null assertion(`!!`)'은 'lazy'한 옵션으로, 일반적인 예외만 발생키시고 구체적인 정보를 제공하지 않는다.  
또한 짧고 간단하기에 개발자가 남용하거나, 잠재적인 'null' 이슈를 놓칠 수 있는 단점이 있다.  
그리고 주로 'Nullable 타입'이지만 실제로는 'null'이 된다고 예상하지 않는 상황에서 사용된다.  
하지만 문제는, 현재 'null'이 예상되지 않더라도 미래에 'null'이 될 수 있음을 고려하지 않아 미래에 발생할 수 있는 오류를 방지하지 못한다.

간단한 예시를 들어, 4개의 'argument' 중 가장 큰 값을 찾는 함수를 생각해보고,
모든 'argument'를 하나의 리스트에 담고 `max`를 사용하여 가장 큰 값을 찾는다고 가정해보자.
`max`의 문제점은 리스트가 빈 경우, 반환 값이 'null' 일 수 있다.
하지만 개발자는 해당 목록이 비어 있을 수 없음을 알고 있기에 'Not-null assertion(`!!`)'을 사용할 가능성이 높다.

```kotlin
fun largestOf(a: Int, b: Int, c: Int, d: Int): Int = 
    listOf(a, b, c, d).max()!!
```

그러나 아무리 간단한 함수라도 'Not-null assertion(`!!`)'을 사용하면 'NPE'가 발생할 수 있다.
예를 들어, 누군가가 이 함수를 리팩토링하여 여러 개의 인자를 받을 수 있도록 하되, 
컬렉션이 비어 있으면 안된다는 중요한 사실을 간과할 경우 아래와 같이 'NPE'가 발생될 수 있다.

```kotlin
fun largestOf(vararg numbers: Int): Int =
    numbers.max()!!

largestOf() // NPE
```

이처럼 'Nullability'에 대한 정보가 은폐되어 있으며, 중요한 상황에서 이를 쉽게 놓칠 수 있다.

변수의 경우도 비슷하다. 어떤 변수가 나중에 설정 되지만 첫 사용 전에 반드시 설정될 것이 분명한 경우가 있을 때,
이런 상황에서 해당 변수를 'null'로 설정하고 'Not-null assertion(`!!`)'을 사용하는 것은 좋지 않은 방법이다.

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

위처럼 프로퍼티를 매번 'unpack'하는 것은 번거롭고, 해당 프로퍼티가 미래에 의미 있는 'null' 값을 가질 수 있는 가능성을 제한한다.
이러한 상황을 적절하게 처리하는 방법은 `lateinit` 또는 `Delgates.notNull`을 [사용](#lateinit-property-and-notnull-delegate)하는 것이다.

미래에 코드가 어떻게 변화할지 누구도 예측할 수 없으며, 'Not-null assertion(`!!`)'을 사용하거나 명시적으로 에러를 발생 시킬 경우, 언젠가 에러가 발생할 것이라 가정해야 한다.
예외는 보통 예상치 못한 상황이나 잘못된 동작을 나타내기 위해 발생시킨다.
하지만 명시적인 에러는 일반적인 'NPE' 보다 훨씬 더 많은 정보를 제공하기에, 대부분의 경우 'Not-null assertion(`!!`)' 보다 더 선호되어야 한다.

'Not-null assertion(`!!`)'이 드물게 의미를 가지는 경우는 대부분 'Nullability'가 표현되지 않은 라이브러리와의 상호 작용에서 발생한다.
그러나 Kotlin을 위해 제대로 설계된 API와 상호 작용 시, 'Not-null assertion(`!!`)'의 사용이 일반적인 표준이 되어선 안된다.

일반적으로 'Not-null assertion(`!!`)'의 사용을 피하는 것이 좋다. 
이러한 권장은 많은 팀들이 'Not-null assertion(`!!`)'를 사용하지 못하게 강하게 정책을 채택하고 있고, 
심지어 `Detekt`와 같은 정적 분석 도구에서도 'Not-null assertion(`!!`)' 사용 시 오류를 발생시키는 설정을 하는 경우도 있다.
이와 같은 접근 방식이 극단적일 수 있다고 생각하지만, 'Not-null assertion(`!!`)' 사용은 'code smell'이 될 수 있다.
생긴 모습도 마치 '주의해라' 또는 '여기에 문제가 있어'라고 알리는 듯한 모습을 하고 있는건 우연이 아닐 것이다.

---

## Avoiding meaningless nullability

'Nullability'는 적절한 처리가 필요하기에 비용이 발생하며, 필요하지 않은 경우 'Nullability'를 피하는 것이 좋다.
'null'은 중요한 메시지를 전달할 수 있지만, 다른 개발자들에게 무의미해 보이는 상황은 피해야 한다.
그렇지 않으면, 안전하지 않은 'Not-null assertion(`!!`)'를 사용하거나, 불필요하게 코드를 복잡하게 만드는 'Null-safe' 처리를 반복하게 될 수 있다.
이로 인해 되도록 의미 없는 'Nullability'는 피하는 것이 좋으며, 이를 위한 방법은 다음과 같다.

- 클래스는 결과를 예상할 수 있는 상황들과 값이 없는 경우를 고려하여, 'nullable 타입' 또는 'sealed class의 sub class'를 반환하는 것 같이 여러 타입의 함수를 제공할 수 있다.
  간단한 예로 `List<T>`의 `get`과 `getOrNull` 같은 함수가 좋은 예시이다.
- 값이 클래스 생성 중이 아닌 사용 전에 확실히 설정되는 경우, `lateinit` 프로퍼티 또는 `Delegate.notNull`을 사용하는 것이 좋다.
- 비어 있는 컬렉션 대신 'null' 반환을 피하라. `List<Int>?` 또는 `Set<String>?`과 같은 컬렉션을 다룰 때, 
  'null'은 비어 있는 컬렉션과는 다른 의미인 컬렉션이 전혀 존재하지 않음을 의미한다. 요소가 없음을 나타내려면 비어 있는 컬렉션을 사용하라.
- 'Nullable enum'과 'None enum' 값은 서로 다른 메시지를 전달한다. 'null'은 별도의 처리가 필요한 특별한 메시지이지만,
  'enum 정의'에는 포함되지 않아, 원하는 모든 'enum'에 추가할 수 있다. 반면 'None'은 'enum 정의'에 포함되어 있어 유효한 값 중 하나로 취급 된다.

---

## lateinit property and notNull delegate

프로젝트에서 클래스 생성 중에 초기화 할 수 없지만 첫 사용 전에 반드시 초기화 될 프로퍼티를 갖는 경우는 일반적이다.   
이런 경우의 일반적인 예로는 'JUnit'의 `@Before`과 같이 다른 함수들 보다 먼저 호출되는 함수에서 설정하는 경우가 있다.

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

위와 같이 프로퍼티들을 필요할 때마다 'Nullable 타입'에서 'Non-nullable 타입'으로 캐스팅하는 것은 권장되지 않는다.
특히, 테스트가 시작되기 전에 이런 값들이 이미 설정 될 것으로 예상한다면, 이는 무의미한 작업이다.
이와 같은 문제에 대한 올바른 해결책은 `lateinit` 수정자를 사용하여 나중에 프로퍼티를 초기화하는 것이다.

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

`lateinit`의 비용은, 초기화되기 전에 값을 얻으려고 시도할 경우 예외가 발생하게 되는데, 이는 개발자들이 원하는 동작이다.
`lateinit`은 첫 사용 전에 반드시 초기화될 것이 확실할 때 사용한다. 만약 이 예상이 틀렸다면 이를 알고 수정 할 수 있을 것이다.
`lateinit`과 'Nullable 타입'의 차이점은 다음과 같다.

- 매번 프로퍼티를 'Non-Nullable 타입'으로 "unpack" 할 필요가 없다.
- 미래에 의미 있는 상황을 나타내기 위해 'null'을 사용해야 할 경우, 쉽게 'Nullable 타입'으로 전환할 수 있다.
- 프로퍼티가 한 번 초기화되면, 초기화되지 않은 상태로 되돌릴 수 없다.

`lateinit`은 첫 사용 전에 프로퍼티가 초기화될 것이 확실한 경우 좋은 방법이다.  
이런 상황은 주로 클래스가 생명주기를 가지고 있고, 생명주기 초기 단계에서 프로퍼티를 설정하여 이후의 메서드에서 사용할 때 다룬다.  
예를 들어, 'Android-Activity의 onCreate', 'iOS-UIViewController의 viewDidLoad' 등이 이에 해당한다. 

`lateinit`을 사용할 수 없는 한 가지 경우가 있다.  
이는 JVM에서 원시 타입으로 취급되는 `Int`, `Long`, `Double`, `Boolean`과 같은 타입으로 속성을 초기화해야 하는 경우 `lateinit`을 사용할 수 없다.
이런 경우에는 원시 타입을 처리하는데 추가적인 'wrapper'와 검증이 필요한 `Delegates.notNull`을 사용 해야한다.

```kotlin
class UserActivity: Activity() {
    private var userAge: Int by Delegates.notNull()
    private var isNotification: Boolean by Delegates.notNull()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        userAge = intent.getIntExtra(USER_AGE_ARG, 0)
        isNotification = intent.getBooleanExtra(NOTIFICATION_ARG, false)
    }
}
```

위와 같은 경우에는 종종 '프로퍼티 위임'을 사용하여 대체 할 수 있다.  
아래와 같이 `onCreate()`에서 인자를 읽는 대신, 해당 프로퍼티를 'lazily'하게 초기화하는 'delegate'를 사용할 수 있다. 

```kotlin
class UserActivity: Activity() {
    private var userAge: Int by arg(USER_AGE_ARG)
    private var isNotification: Boolean by arg(NOTIFICATION_ARG)
}
```