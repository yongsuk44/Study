객체를 정의하고 생성하는 방법을 지정할 때 가장 많이 사용되는 방법은 'primary constructor'를 사용하는 것이다.

```kotlin
class User(var name: String, var surname: String)
val user = User("John", "Doe")
```

이처럼, 'primary constructor'는 매우 편리할 뿐만 아니라 대부분의 경우 객체를 빌드하는데 매우 좋은 관행이다.

'primary constructor'는 객체의 초기 상태를 결정하는 인자를 전달해야 하는 경우가 흔하며, 이는 데이터를 나타내는 데이터 모델 객체를 예시로 들 수 있다.
데이터 모델 객체는 생성 시 모든 상태가 생성자를 통해 초기화되며, 이후에는 해당 상태가 객체의 프로퍼티로서 유지된다.

```kotlin
data class Student(
    val name: String,
    val surname: String,
    val age: Int
)
```

다음은 인덱싱된 따옴표 시퀀스를 표시하기 위한 프레전터를 생성하는 또 다른 예시이다.
여기서는 'primary constructor'를 사용하여 의존성을 주입한다.

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

'QuotationPresenter'는 'primary constructor'에 선언된것 보다 더 많은 프로퍼티를 가지고 있음에 주목하자.
여기서 'nextQuoteId' 프로퍼티는 항상 -1로 초기화되는 특징을 가지며, 이는 객체의 초기 상태를 설정하는데 있어 'default value' 또는 'primary constructor' 파라미터를 활용하는 것이 타당하다는 점을 보여준다.

대부분의 경우에서 'primary constructor'가 좋은 방식이라는 것을 이해하기 위해, 먼저 생성자와 관련된 일반적인 Java 패턴을 고려해보자.

- telescoping constructor pattern
- builder pattern

위 2가지 패턴들이 해결하는 문제와 Kotlin이 제공하는 더 나은 방식에 대해 알아보자.

---

## Telescoping constructor pattern

'telescoping constructor pattern'은 객체를 생성할 때 여러 생성자를 제공하여 다양한 파라미터 조합을 사용할 수 있게하는 디자인 패턴이다.

```kotlin
class Pizza {
    val size: String
    val cheese: Int
    val olives: Int
    val bacon: Int
    
    constructor(size: String, cheese: Int, olives: Int, bacon: Int) {
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

그러나, Kotlin에서는 'default argument'를 사용할 수 있기에 위와 같은 코드는 의미가 없다.

```kotlin
class Pizze(
    val size: String,
    val cheese: Int = 0,
    val olives: Int = 0,
    val bacon: Int = 0
)
```

'default argument'를 사용하면, 개발자는 필요한 파라미터만 선택적으로 제공할 수 있어 코드를 더 깔끔하고 간결하게 만들 수 있다.

```kotlin
val myFavorite = Pizza("L", olives = 2)
```

또한, 'named argument'를 사용함으로써 파라미터의 인자의 순서를 자유롭게 변경할 수 있다.

```kotlin
val villagePizza = Pizza(size = "L", olives = 2, cheese = 1)
```

정리하면, 'default argument'는 다음과 같은 이유로 'telescoping constructor pattern'보다 더 강력하다.

- 원하는 인자들의 부분집합을 설정할 수 있다.
- 인자를 어떠한 순서로든 제공할 수 있다.
- 'named argument'를 통해 각 값이 무엇을 의미하는지 명확하게 할 수 있다.

마지막 이유는 매우 주용하며, 다음과 같이 객체를 생성한다고 생각해보자. 

```kotlin
val villagePizza = Pizza("L", 1, 2, 3)
```

코드가 간결하지만, 명확한가? 아마도 'Pizza' 클래스를 선언한 사람조차도 'bacon'이 어디에 위치하는지, 'cheese'가 어디에 위치하는지 기억하지 못할 것이다.
물론, IDE에서 설명을 볼 수 있지만, Github에서 코드를 읽는 상황에서는 어떨까?

이처럼 인자가 명확하지 않을 때는 'named argument'를 사용하여 각 인자의 이름을 명시적으로 지정해주는 것이 좋다.

```kotlin
val villagePizza = Pizza(size = "L", cheese = 1, olives = 2, bacon = 3)
```

보다시피, 'default argument'가 있는 생성자는 'telescoping constructor pattern'을 능가한다.
또한, Java에서 더 많이 사용되는 생성자 패턴 중 하나인 'builder pattern'과 비교해도 'default argument'가 더 강력하다.

---

## Builder pattern

Java에서는 'named paramter', 'default argument'를 허용하지 않기에, 주로 'builder pattern'을 사용한다.
이를 통해, 다음과 같은 작업을 할 수 있다.

- 파라미터에 이름을 붙여, 각 파라미터가 무엇을 의미하는지 쉽게 이해할 수 있다.
- 파라미터를 어떤 순서로든 지정할 수 있어, 객체 생성 시 필요한 파라미터만 선택적으로 제공할 수 있다.
- 'default value'를 가질 수 있어, 선택적으로 인자를 설정할 수 있다.

다음은 Kotlin에서 정의된 'Builder'의 예시이다.

```kotlin
class Pizza private constructor(
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
        
        fun build() = Pizza(size, cheese, olives, bacon)
    }
}
```

'builder pattern'을 통해, 다음과 같이 원하는대로 파라미터를 설정할 수 있다.

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

이미 언급했듯이, 이 두 가지 장점은 Kotlin의 'default argument'와 'named argument'로 완전히 대체될 수 있다.

```kotlin
val villagePizza = Pizza(size = "L", cheese = 1, olives = 2, bacon = 3)
```

간단하게 'named argument'와 'builder'를 비교해보면, 'named argument'가 'builder'보다 더 많은 이점을 갖는다는 것을 알 수 있습니다.

'default argument'가 있는 생성자나 팩토리 메서드는 'builder pattern'을 구현하는 것 보다 더 간단하다.
이는 코드를 구현하는 개발자와 코드를 읽는 사람 모두에게 시간을 절약해준다.
또한, 'builder pattern'의 구현은 시간이 많이 소요되며, 수정하는 것 또한 어렵다.
예를 들어, 파라미터의 이름이 변경되면 해당 파라미터를 설정하는데 사용된 함수의 이름 변경뿐만 아니라, 이 함수 내의 파라미터 이름, 함수의 본문, 이를 유지하는 내부 필드, 비공개 생성자 내의 파라미터 이름 등을 변경해야 한다.

객체가 어떻게 구성되는지, 어떻게 유지되는지, 어떻게 상호작용하는지 등을 보고 싶을 떄, 
'builder pattern'은 객체 생성 로직이 빌더 클래스 전체에 분산되어 있어 확인하기 어려운 반면, 
'named argument'와 'default argument'를 사용하는 방식은 객체의 생성 과정을 한 곳에 집중시켜 이해하기 쉽다.

'primary constructor'는 내장된 개념이어서 자연스럽고 익숙하게 사용할 수 있다. 반면, 'builder pattern'은 인위적인 개념이기에 이에 대한 지식이 요구된다.
예를 들어, 객체 생성을 위해 여러 단계를 거치고, 마지막에 반드시 'build()'를 호출해야 한다. 
만약 이를 잊어버리거나 생략하면, 객체가 제대로 생성하지 않는 문제가 발생할 수 있다.

드문 문제이지만, 대부분의 'builder pattern'의 프로퍼티는 가변 요소이기에 여러 스레드가 동시에 builder 객체에 접근하여 프로퍼티를 변경하려고 하면 동시성 문제가 발생할 수 있다.
반면, Kotlin에서 함수 파라미터는 항상 불변이기에 한 번 값을 할당하면 그 값이 변경되지 않는다.
따라서 'builder pattern'의 'build()'를 'Thread-safe'하게 구현하는 것이 더 복잡하다.

이러한 장점들이 있어서 'builder pattern' 대신 생성자를 사용해야 한다는 의미는 아니다.  
다음은 'builder pattern'이 더 유용할 수 있는 시나리오이다.

'builder pattern'은 객체를 구성할 때, 특정 메서드를 통해 필요한 값을 설정할 수 있으며(setPositiveButton, setNegativeButton, and addRoute), 이를 통해 여러 값을 객체에 추가할 수 있도록 해준다.(addRoute)

```kotlin
val dialog = AlertDialog.Builder(context)
    .setMessage("Are you sure you want to delete the file?")
    .setPositiveButton("Delete") { dialog, id ->
        // Yes, delete the file
    }
    .setNegativeButton("Cancel") { dialog, id ->
        // No, don't delete the file
    }
    .create()

val router = Router.Builder()
    .addRoute(path = "/home", ::showHome)
    .addRoute(path = "/users", ::showUsers)
    .build()
```

만약, 비슷한 동작을 생성자를 통해 하기 위해서는 단일 인자에서 더 많은 데이터를 보유할 수 있는 특수 타입을 도입해야 한다.

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

위와 같은 표기법은 일반적으로 Kotlin 커뮤니티에서 부정적으로 받아들여지며, 이러한 경우에는 다음과 같이 DSL 사용을 선호하는 경향이 있다.

```kotlin
val dialog = context.alert("Are you sure you want to delete the file?") {
    positiveButton("Delete") { }
    negativeButton("Cancel") { }
}

val route = router {
    "/home" directsTo ::showHome
    "/users" directsTo ::showUsers
}
```

이런 종류의 DSL은 더 많은 유연성과 깔끔한 표기법을 제공하기에, 일반적으로 'builder pattern' 보다 선호된다.
물론, DSL을 만드는 것이 더 어려운 것이 사실이지만, 'builder'를 만드는 것도 어려울 것이다.
만약 더 나은 표기법을 위해 더 많은 비용을 투자하기로 결정했다면, 덜 명확한 정의의 비용을 감수하더라도 한 단계 더 나아가는 것이 좋다.
이 대가로 개발자들은 더 많은 유연성과 가독성을 얻게 될것이다.

'builder pattern'의 또 다른 장점은 팩토리로 사용될 수 있다는 것이다.  
예를 들어, 애플리케이션의 'default dialog'처럼 부분적으로 구성된 객체를 다른 컨텍스트나 상황에 맞게 구성할 수 있다.

```kotlin
fun Context.makeDefaultDialogBuilder() = 
    AlertDialog.Builder(this)
        .setIcon(R.drawable.ic_dialog)
        .setTitle(R.string.dialog_title)
        .setOnDismissListener { it.cancel() }
```

이를 생성자나 팩토리 함수에서 유사한 기능을 가지려면, 하나의 인자만 받고 나머지 인자를 받기 위한 또 다른 함수를 반환하는 기법인 'currying'이 필요하겠지만 Kotlin에서는 지원되지 않는다. 
이에 대한 대안으로, 객체 구성을 'data class'로 관리하고, 'copy'를 통해 기존 객체를 수정할 수 있다.

```kotlin
data class DialogConfig(
    @DrawableRes val icon: Int = -1,
    @StringRes val title: Int = -1,
    val onDismiss: (() -> Unit)? = null
)

fun makeDefaultDialogConfig() = DialogConfig(
    icon = R.drawable.ic_dialog,
    title = R.string.dialog_title,
    onDismiss = { it.cancel() }
)
```

비록 두 방식이 선택지로 거의 고려되지 않지만, 예를 들어 애플리케이션의 'default dialog'를 정의하고 싶다면, 함수를 통해 모든 커스터마이징 요소를 'optional argument'로 전달하는 방식은 더 많은 제어권과 유연성을 제공한다.
이것이 바로 'builder pattern'의 이점 중 하나지만, 이러한 이점은 사소하게 여겨지는 경우가 많다.

결국에 'builder pattern'은 Kotlin에서 최선의 선택지가 되기 힘들다. 그러나 때때로 다음과 같은 이유로 선택된다.

- 다른 언어로 작성된 라이브러리와 코드의 일관성을 유지하기 위해, 해당 라이브러리들이 'builder pattern'을 사용하는 경우
- 다른 언어에서 기본 인자나 DSL을 지원하지 않는 경우, 다른 언어에서 쉽게 사용할 수 있도록 API를 설계할 때

그 외에는 'default argument'를 가지는 'primary constructor'나 DSL을 선호한다.