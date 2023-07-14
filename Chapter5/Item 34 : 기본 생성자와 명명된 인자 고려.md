# Item 34: 기본 생성자와 선택적인 명명된 인자 고려

객체를 정의하고 그것이 어떻게 생성될 수 있는지 명시할 때, 대부분의 경우 기본 생성자를 사용하는 것이 좋습니다.

```kotlin
class User(var name: String, var surname: String)
val user = User("John", "Doe")
```

기본 생성자는 간단하며 객체 생성 시 대부분의 상황에서 사용될 수 있습니다.

데이터 모델 객체들은 생성자를 이용해 전체 상태를 초기화하고 그 상태를 프로퍼티로 유지 합니다.

```kotlin
data class Student(
    val name: String,
    val surname: String,
    val age: Int
)
```

인덱싱된 `Quotes`를 표시하기 위해 기본 생성자를 통해 의존성이 주입된 Presenter를 만드는 예시입니다.

```kotlin
class QuotesPresenter(
    private val quotesRepository: QuotesRepository,
    private val view: QuotesView
) {
    private var nextQuoteId = -1
    
    fun onStart() { 
        onNext()
    }
    
    fun onNext() {
        nextQuoteId = (nextQuoteId + 1) % quotesRepository.quotesNumber
        
        val quote = quotesRepository.getQuote(nextQuoteId)
        view.showQuote(quote)
    }
}
```

`QuotationPresenter`는 기본 생성자에 선언된 것보다 더 많은 속성을 가지고 있는것을 알 수 있습니다.

여기서 `nextQuoteId`는 항상 -1로 초기화되는 프로퍼티입니다.   
초기 상태가 기본값 또는 기본 생성자의 인자를 사용하여 설정될 때 이렇게 선언되는 것은 상관 없습니다.

생성자와 관련된 일반적인 Java 패턴을 통해 기본 생성자가 대부분의 경우에 사용되는지를 알 수 있습니다.

---

## Telescoping constructor pattern

텔레 스코핑 생성자 패턴(telescoping constructor pattern)은 간단히 설명하면 다양한 인자 조합들을 위한 생성자 집합입니다.

```kotlin
class Pizza {
    val size: String,
    val cheese: Int,
    val olives: Int,
    val bacon: Int
    
    constructor(size: STring, cheese: Int, olives: Int, bacon: Int) {
        this.size = size
        this.cheese = cheese
        this.olives = olives
        this.bacon = bacon
    }
    
    constructor(size: String, cheese: Int, olives: Int) : this(size, cheese, olives, 0)
    constructor(size: String, cheese: Int) : this(size, cheese, 0)
    constructor(size: String) : this(size, 0)
}
```

그러나, Kotlin에서는 기본인자를 사용할 수 있기 때문에 위와 같은 코드가 필요가 없습니다.

```kotlin
class Pizze(
    val size: String,
    val cheese: Int = 0,
    val olives: Int = 0,
    val bacon: Int = 0
)
```

Kotlin의 기본 인자는 다음과 같은 이유로 텔레스코핑 생성자보다 더 좋습니다.

• 원하는 기본 인자들의 임의의 부분집합을 설정할 수 있습니다.
• 인자를 어떠한 순서로든 제공할 수 있습니다.
• 각 값이 무엇을 의미하는지 명확하게 하기 위해 인자를 명시적으로 명명할 수 있습니다.

마지막 포인트는 특히 중요합니다. 다음과 같은 객체 생성을 생각해봅시다: `val villagePizza = Pizza("L", 1, 2, 3)`

---

## Builder pattern

Java에서는 파라미터에 이름을 지정하거나 기본값을 할당하는 것이 불가능하여 Java 개발자들은 여러 상황에서 빌더 패턴(Builder Pattern)을 사용하였습니다.

빌더 패턴은 아래와 같은 유연성을 제공합니다.

- 파라미터에 명확한 이름을 붙여 코드의 가독성을 높입니다.
- 파라미터를 원하는 순서로 지정할 수 있어 코드 작성의 자유도를 높입니다.
- 파라미터에 기본값을 설정할 수 있어 선택적인 값을 처리하는데 용이합니다.

아래 예제는 Kotlin에서 빌더패턴을 이용하여 클래스를 정의한 것입니다.

```kotlin
class Pizza(
    val size: String,
    val cheese: Int, 
    val olives: Int,
    val bacon: Int
) {
    class Builder(private val size: String) { 
        
        private var cheese: Int = 0
        private var olives: Int = 0
        private var bacon: Int = 0
        
        fun setCheese(value: Int): Builder = apply { cheese = value }
        fun setOlives(value: Int): Builder = apply { olives = value }
        fun setBacon(value: Int): Builder = apply { bacon = value }
        
        fun build() = Pizza(size, cheese, olives, bacon) }
}
```

빌더 패턴을 사용하면 파라미터를 사용자 정의로 설정하고 이름을 통해 이를 수행할 수 있습니다.

```kotlin
val myFavorite = Pizza.Builder("L")
    .setOlives(2)
    .build()

val pizza = Pizza.Builder("L")
    .setCheese(1)
    .setOlives(2)
    .setBacon(3)
    .build()
```

그러나, Kotlin에서는 기본 인자와 이름있는 인자를 사용할 수 있기 때문에 빌더 패턴을 사용할 필요가 없습니다.

```kotlin
val villagePizza = Pizza(size = "L", cheese = 1, olives = 2, bacon = 3)
```

간단하게 2가지 방법을 비교하면, 명명된 파라미터가 빌더 패턴보다 어떤 이점을 갖는지 알 수 있습니다.

- 디폴트 인자를 갖는 생성자나 팩토리 메서드는 빌더 패턴을 구현하는 것보다 간단합니다.
- 객체가 어떻게 생성되는지 보기 위해선 빌더 클래스를 훑어보는 대신, 하나의 메서드에서 모든 것을 확인할 수 있어 깔끔합니다.
- 기본 생성자는 내장된 개념이라는 점에서 빌더 패턴이 요구하는 추가 지식 없이 사용하기가 더 단순합니다.
- 함수 파라미터는 항상 불변이며, 대부분의 빌더에서 속성은 변경 가능하므로, Thread-safe한 빌드 함수를 빌더에 구현하는 것이 어렵습니다.

이는 항상 생성자를 빌더 대신 사용해야 한다는 의미가 아닙니다.

빌더 패턴의 다른 장점이 효과적으로 작동하는 경우도 있습니다.

빌더 패턴은 이름에 대한 값을 설정하도록 요구하고, 값을 집계하는 것을 허용합니다.
(setPositiveButton, setNegativeButton, and addRoute)

```kotlin
val dialog = AlertDialog.Builder(context)
    .setMessage("Are you sure you want to delete the file?")
    .setPositiveButton("Delete") { dialog, _ -> }
    .setNegativeButton("Cancel") { dialog, _ -> }
    .create()

val router = Router.Builder()
    .addRoute(path = "/home", ::showHome)
    .addRoute(path = "/users", ::showUsers)
    .build()
```

생성자로 이러한 동작을 달성하려면, 하나의 인자에서 더 많은 데이터를 보유하도록 특별한 타입을 도입해야 합니다.

```kotlin
val dialog = AlertDialog(
    context = context,
    message = "Are you sure you want to delete the file?",
    positiveButton = Button("Delete") { dialog, _ -> },
    negativeButton = Button("Cancel") { dialog, _ -> }
)

val router = Router(
    routes = listOf(
        Route(path = "/home", handler = ::showHome),
        Route(path = "/users", handler = ::showUsers)
    )
)
```

이러한 표기법은 일반적으로 Kotlin 커뮤니티에서 부정적이며, 이런 경우 대부분은 DSL(Domain Specific Language)을 사용하는 것을 선호합니다.

```kotlin
val dialog = context.alert("Are you sure you want to delete the file?") {
    positiveButton("Delete") { dialog, _ -> }
    negativeButton("Cancel") { dialog, _ -> }
}

val route = router {
    "/home" directsTo ::showHome
    "/users" directsTo ::showUsers
}
```

이런 종류의 DSL은 더 많은 유연성과 깔끔한 표기법을 제공하기에 빌더보다 더 선호되는 경향이 있습니다.

빌더 패턴의 또 다른 장점은 팩토리로 사용될 수 있다는 점입니다. 이는 부분적으로 채워져 전달 될 수 있습니다.

```kotlin
fun Context.makeDefaultDialogBuilder() = 
    AlertDialog.Builder(this)
        .setIcon(R.drawable.ic_dialog)
        .setTitle(R.string.dialog_title)
        .setOnDismissListener { /* ... */ }
```

생성자 또는 팩토리 메서드에서 이와 유사한 가능성을 가지려면, 커링이 필요한데 Kotlin에서는 이를 지원하지 않습니다.

```kotlin
data class DialogConfig(
    @DrawableRes val icon: Int = -1,
    @StringRes val title: Int = -1,
    val onDismiss: () -> Unit? = null
)

fun makeDefaultDialogConfig() = DialogConfig(
    icon = R.drawable.ic_dialog,
    title = R.string.dialog_title,
    onDismiss = { /* ... */ }
)
```

결론적으로 빌더 패턴은 Kotlin에서 드물게 사용되며, 아래의 경우에 사용합니다.
- 빌더 패턴을 사용한 다른 언어로 작성된 라이브러리와 코드를 일관성 있게 만드는 경우
- 기본 인자나 DSL을 지원하지 않는 다른 언어에서 쉽게 사용될 수 있도록 API 설계 시

그 외에는 대부분 기본 생성자 혹은 기본 인자를 선호하거나, 표현력 있는 DSL을 선호합니다.