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
