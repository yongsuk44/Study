'Item 27'에서 메시지를 표시하는 함수를 다시 살펴보자.

```kotlin
enum class MessageLength { SHORT, LONG }

fun Context.showMessage(
    message: String,
    length: MessageLength = MessageLength.LONG
) {
    val toastLength = when (length) {
        SHORT -> Length.SHORT
        LONG -> Length.LONG
    }

    Toast.makeText(this, message, toastLength).show()
}
```

위 코드는 메시지 표시 방법을 자유롭게 변경하기 위해 추출하였다.   
그러나, 이 함수는 다른 개발자가 읽었을 때 항상 토스트를 표시한다고 가정할 수 있습니다.  
이는 구체적인 메시지 타입을 제안하지 않는 방식으로 이름을 지음으로써, 원래 의도했던 것과는 반대가 된다.

이처럼 기대하는 동작을 명확히 하기 위해서는 의미 있는 KDoc 주석을 추가하는 것이 좋다.

```kotlin
enum class MessageLength { SHORT, LONG }

/**
 * Universal way for the project to display a short message to a user
 *
 * @param message The text that should be shown to the user
 * @param duration How long to display the message
 */
fun Context.showMessage(
    message: String,
    duration: MessageLength = MessageLength.LONG
) {
    val toastLength = when (duration) {
        SHORT -> Length.SHORT
        LONG -> Length.LONG
    }

    Toast.makeText(this, message, toastLength).show()
}
```

대부분의 경우, 코드나 함수의 이름만으로는 그 기능이나 목적을 완전히 전달하기 어렵다.  
예를 들어, 'powerset'은 잘 정의된 수학적 개념이지만, 널리 알려져 있고 해석이 명확하지 않기에 추가적인 설명이 필요하다.

```kotlin
/**
 * Powerset returns a set of all subsets of the receiver
 * including itself and the empty set
 */
fun <T> Collection<T>.powerset(): Set<Set<T>> =
    if (this.isEmpty()) {
        setOf(emptySet())
    } else {
        take(this.size.minus(1))
            .powerset()
            .let { it + it.map { it + this.last() } }
    }
```

주석에 대한 설명은 개발자가 내부 구현을 개선하거나 변경할 수 있는 유연성을 제공한다.

위 'powerset'은 요소들의 순서를 구체적으로 지정하지 않았지만, 추상화 뒤에 구현이 숨겨져 있기에 사용자들 모르게 최적화 될 수 있다.
이러한 이유로 사용자 입장에서는 내부 구현에 의존하지 않고, 해당 추상화에 의존해야 한다.

```kotlin
/**
 * Powerset returns a set of all subsets of the receiver
 * including itself and the empty set
 */
fun <T> Collection<T>.powerset(): Set<Set<T>> =
    powerset(this, setOf(setOf()))

private tailrec fun <T> powerset(
    left: Collection<T>,
    acc: Set<Set<T>>
): Set<Set<T>> =
    if (left.isEmpty()) {
        acc
    } else {
        powerset(
            left = left.drop(1),
            acc = acc + acc.map { it + left.first() }
        )
    }
```

일반적인 문제는 함수의 동작이 문서화되어 있지 않고, 요소들의 이름이 그 기능이나 목적을 명확하게 전달하지 못한다면,
개발자들은 의도한 추상화에 의존하지 않고, 내부 구현에 의존하게 된다. 이러한 문제는 기대할 수 있는 동작을 명확히 설명함으로써 해결할 수 있다.

## Contract

제공자가 어떤 동작을 설명할 때, 사용자는 그 설명을 약속으로 간주하고, 이를 기준으로 자신이 할 수 있는 기대치를 조정한다.
이처럼 기대되는 동작들을 요소의 'contract'라고 한다.
실제 계약에서 상대방이 계약을 지켜주길 기대하는 것처럼, 동일하게 사용자들은 'contract'이 안정화되면 제공자가 이를 잘 유지할 것이라 기대한다.

'contract'을 정의하는 것이 무서워 보일 수 있지만, 잘 명시된 'contract'이 있다면 양측 모두에게 다음과 같은 이점이 있다.

1. 제공자는 클래스가 어떻게 사용되는지 걱정하지 않아도 되고, 사용자는 내구 구현이 어떻게 되어있는지 걱정 할 필요가 없다.
2. 사용자는 구현에 대한 지식이 없어도 'contract'를 통해 기능을 사용할 수 있고, 제공자는 'contract'이 위반되지 않는 선에서 내부 구현을 자유롭게 변경할 수 있다.
3. 양측 모두 추상회된 'contract'에 의존함으로써, 독립적으로 작업이 가능하다.

이러한 이유로, 'contract'이 지켜지는 한 모든 것이 잘 동작할 것이며, 이는 양측 모두에게 편안함과 자율성을 준다.

만약 'contract'이 설정되지 않았다면, 사용자들은 무엇을 할 수 있고 할 수 없는지 알 수 없어, 내부 구현을 의존하게 될 것이다.
이럴 경우 제공자는 사용자들이 어떤 내부 구현을 의존하는지 모르기에, 구현 사항을 변경할 때 마다 기존 사용자 코드를 깨뜨릴 수 있다.
이러한 이유로 'contract'을 명시하는 것이 중요하다.

## Defining a contract

'contract'를 정의하는 방법은 다양하며, 다음과 같다.

- Names : 이름이 일반적인 개념과 연결될 때, 해당 요소가 이 개념과 일관되게 행동할 것이라 기대한다.
  예를 들어, 'sum' 메서드는 잘 정의된 수학적 개념이기에, 어떤 행동을 할지 알기 위해 주석을 읽을 필요가 없다.
- Comments and Documentation : 필요한 모든 것을 설명할 수 있기에 가장 효과적인 방법이다.
- Types : 타입은 객체에 대한 많은 정보를 제공한다.
  때때로 각 타입은 잘 정의된 메서드의 집합을 알려주며, 일부 타입은 객체를 사용하기 전에 초기 설정이나 준비 작업을 문서에 명시하기도 한다.

## Do we need comments?

역사를 돌아보면, 커뮤니티 내에서 의견들이 어떻게 변화되는지 지켜보는 것은 재미있는 일이다.  
초기 Java 시대에는 코드에 대한 설명을 주석으로 상세하게 기술하는 문해 프로그래밍(literate programming)이 매우 인기 있었다.
그러나 10년이 지나고, 주석에 대한 강한 비판과 주석을 생략하고 읽기 쉬운 코드 작성에 집중해야 한다는 주장이 강해졌다.('Clean Code')

어떠한 극단적인 입장도 좋지 않다.
가독성 있는 코드를 작성하는 것도 중요하지만, 요소(함수나 클래스) 앞에 붙는 주석이 해당 요소를 더 높은 수준에서 설명하고, 'contract'를 설정하는데 더 도움이 될 수 있다.
게다가, 요즘 주석은 종종 문서를 자동으로 생성하는데 사용되며, 이런 문서는 프로젝트의 'source of truth'로 간주된다.

물론, 모든 상황에 주석이 필요한 것은 아니다.
예를 들어, 많은 함수들은 자체적으로 이해할 수 있기에 특별한 설명이 필요 없다.
아래 'product'가 프로그래머들에게 알려진 명확한 수학적 개념이라고 가정하면 주석 없이 충분히 이해할 수 있다.

```kotlin
fun List<Int>.product(): Int = fold(1) { acc, i -> acc * i }
```

뻔한 주석은 오히려 방해가 될 수 있다.
함수 이름과 파라미터가 명확하게 표현된 내용을 설명하는 주석은 작성하지 않는 것이 좋다.
아래는 메서드의 이름과 파라미터 타입으로 어떤 동작을 하는지 유추할 수 있기에, 불필요한 코멘트를 보여주는 예시이다.

```kotlin
// Product of all numbers in a list
fun List<Int>.product(): Int = fold(1) { acc, i -> acc * i }
```

또한 코드를 정리할 필요가 있을 때, 구현에서 주석 대신 함수로 추출하는 것이 좋다.

```kotlin
fun update() {
    // Update users
    for (user in users) user.update()

    // Update books
    for (book in books) updateBook(book)
}
```

위 'update' 함수는 명확하게 추출 가능한 부분들로 구성되어 있으며, 주석은 이런 부분들이 다른 설명으로 대체 될 수 있음을 제안한다.
따라서 이런 부분들을 별도의 추상화, 예를 들어 메서드로 추출하는 것이 더 좋다. 그리고 그 이름만으로도 무엇을 의미하는지 충분히 설명할 수 있다.

```kotlin
fun update() {
    updateBooks()
    updateUsers()
}

private fun updateBooks() {
    for (book in books) {
        updateBook(book)
    }
}

private fun updateUsers() {
    for (user in users) {
        user.update()
    }

}
```

그러나, 주석이 유용하고 중요한 경우도 있다.
예를 들어, Kotlin 표준 라이브러리의 모든 public 함수를 살펴보면, 잘 정의된 'contract'를 통해 높은 자율성을 제공하고 있다.
아래 'listOf'를 살펴보자.

```kotlin
/**
 * Returns a new read-only list of given elements.
 * The returned list is serializable (JVM).
 * @sample samples.collections.Collections.Lists.readOnlyList
 */
public fun <T> listOf(vararg elements: T): List<T> =
    if (elements.size > 0) elements.asList()
    else emptyList()
```

위 함수는 단지 반환되는 List가 읽기 전용이며, JVM에서 직렬화가 가능하다는 것만을 약속하고 있다.  
또한, 리스트가 불변할 필요가 없으며, 반환되는 구체적인 타입에 대한 어떠한 것도 약속을 하지 않고 있다.
이러한 'contract'는 많은 유연성을 제공하기에 대부분의 Kotlin 개발자들이 만족스럽게 사용할 수 있을 것이다.
마지막으로, 사용 예시를 제공하여 해당 요소의 사용법을 알려주어 어떻게 사용될 수 있는지 알려준다.

## KDoc format

주석으로 함수를 문서화할 때, 해당 주석을 작성하는 공식적인 포맷을 'KDoc'이라고 한다.

모든 'KDoc' 주석은 /**로 시작하여 */로 끝나며, 내부적으로 모든 줄은 일반적으로 *로 시작한다.  
여기에 있는 설명들은 KDoc 마크다운으로 작성된다.

KDoc 주석 구조는 다음과 같다.

- 첫 번째 단락은 해당 요소의 '요약 설명'
- 두 번째 부분은 해당 요소의 '자세한 설명'
- 그 다음 모든 줄은 태그로 시작하며, 해당 태그들은 설명하고자 하는 요소를 참조하는데 사용

KDoc에서 지원하는 태그들은 다음과 같다.

| 태그                                      | 설명                                                                                          |
|-----------------------------------------|---------------------------------------------------------------------------------------------|
| `@param <name>`                         | - 함수의 'value 파라미터', '클래스', '프로퍼티', '타입 파라미터'를 문서화한다. <br/> - 파라미터의 용도와 기대되는 값의 종류에 대해 설명한다. |
| `@return`                               | - 함수의 반환 값에 대해 문서화한다. <br/>- 함수가 반환하는 값의 타입과 의미를 설명한다.                                      |
| `@constructor`                          | - 클래스의 기본 생성자를 문서화한다. <br/>- 생성자의 역할과 사용법에 대해 설명한다.                                         |
| `@receiver`                             | - 확장 함수의 수신 객체를 문서화한다. <br/>- 확장 함수가 적용되는 대상의 타입에 대해 설명한다.                                  |
| `@property <name>`                      | - 지정된 이름을 가진 클래스의 프로퍼티를 문서화한다. <br/>- 기본 생성자에 정의된 프로퍼티를 문서화 할 때 사용된다.                       |
| `@throws <class> or @exception <class>` | - 메서드에 의해 발생될 수 있는 예외를 문서화한다. <br/>- 예외 타입과 발생 조건에 대해 설명한다.                                 |
| `@sample <identifier>`                  | - 현재 요소를 사용하는 방법의 예제를 보여주기 위해 지정된 함수의 본문을 현재 요소의 문서화에 포함시킨다.                                |
| `@see <identifier>`                     | - 지정된 클래스나 메서드로의 링크를 추가한다. <br/>- 관련 내용이나 추가 정보를 제공하는 요소를 참조할 때 유용한다.                       |
| `@author`                               | - 문서화되는 요소의 저자를 명시한다. <br/>- 코드 작성자나 문서 작성자의 이름을 기록할 때 사용된다.                                |
| `@since`                                | - 문서화되는 요소가 도입된 소프트웨어 버전을 명시한다. <br/>- 해당 요소가 언제부터 사용 가능한지를 나타낸다.                           |
| `@suppress`                             | - 생성된 문서에서 특정 요소를 제외시킨다. <br/>- 공식 API의 일부는 아니지만 외부적으로 보여져야 하는 요소들에 대해 사용된다.                |

설명과 태그를 설명하는 텍스트에서 클래스, 메서드, 프로퍼티, 파라미터 등을 링크 할 수 있다.
링크는 대괄호 안에 있거나, 연결된 요소의 이름과 다른 설명을 원할 때는 이중 대괄호를 사용한다. 

```kotlin
/**
 * This is an e.g description linking to [element1],
 * [com.package.SomeClass.element2] and [this element with custom description][element3]
 */
```

이런 모든 태그들은 Kotlin 문서 생성 도구에 의해 이해되며, 공식 도구는 'Dokka'라고 부른다.
이들은 온라인에서 게시되고 외부 사용자들에게 제시될 수 있는 문서 파일들을 생성한다. 
다음은 설명이 간략하게 정리된 예시 문서이다.

```kotlin
/**
 * Immutable tree data structure.
 * 
 * Class represents immutable tree having from 1 to infinite number of elements.
 * In the tree we hold elements on each node and nodes can have left and right subtrees...
 * 
 * @param T the type of elements this tree holds.
 * @property value the value kept in this node of the tree
 * @property left the left subtree
 * @property right the right subtree
 */
class  Tree<T>(
    val value: T,
    val left: Tree<T>? = null,
    val right: Tree<T>? = null
) {

  /**
   * Creates a new tree based on the current but with [element] added
   * @return newly created tree with additional element
   */
  override fun plus(element: T): Tree { ... }
}
```

모든 것을 설명할 필요는 없다. 
문서는 짧고, 명확하지 않은 부분을 정확하게 설명하는 문서가 가장 좋은 문서다. 

---

## Type system and expectations

객체의 타입 계층구조는 해당 객체의 기능과 행동에 대한 중요한 정보를 제공한다.  
인터페이스와 클래스는 구현해야 하는 메서드 뿐만 아니라, 행동 방식과 상태 관리 등 특정 기대치를 정의할 수 있다. 

만약, 클래스가 어떤 기대치를 약속했다면, 해당 클래스의 하위 클래스들 또한 이를 보장해야 한다.
이런 약속은 '리스코프 치환 원칙'으로 알려져 있으며, OOP에서 가장 중요한 규칙 중 하나이다. 
일반적으로 "S가 T의 하위 타입이면, T 타입의 객체는 프로그램의 원하는 속성을 변경하지 않고 S 타입의 객체로 대체할 수 있어야 한다"로 알려져 있다.

이 규칙이 중요한 이유는 모든 클래스는 상위 클래스가 될 수 있고, 만약 상위 클래스가 되어야 하는 역할을 충실히 수행하지 못하는 경우 예상치 못한 문제가 발생할 수 있다.
프로그래밍에서, 하위 클래스는 상위 클래스의 계약을 이행해야 한다.

이 규칙에서 중요한 점 중 하나는 오픈 함수에 대한 'contract'를 적절하게 명시해야 한다는 것이다.  
예를 들어, 다음과 같은 인터페이스를 사용하여 자동차를 표현할 수 있을 것이다.

```kotlin
interface Car {
    fun setWheelPosition(angle: Float)
    fun setBreakPedal(pressure: Double)
    fun setGasPedal(pressure: Double)
}

class GasolineCar: Car {
    // ...
}

class GasCar: Car {
    // ...
}

class ElectricCar: Car {
    // ...
}
```

'Car' 인터페이스의 문제점은 많은 질문이 생긴다는 것이다.

"'setWheelPosition'에서 'angle'은 무엇을 의미하고, 어떤 단위로 측정되는지?"  
"가스와 브레이크 페달이 무엇을 하는지 명확하지 않은 경우는 어떻게 해야 하는가?"

또한, 'Car' 타입의 인스턴스를 사용하는 사람들은 이를 어떻게 사용해야 하는지 알아야 하며, 모든 브랜드가 'Car'로 사용될 때 비슷하게 동작해야 한다.
이러한 우려사항들을 다음과 같이 문서화하여 해결할 수 있다.

```kotlin
interface Car {

    /**
     * Changes car direction.
     * 
     * @param angle Represents position of wheels in radians relative to the car axis.
     * 0 means driving straight, pi / 2 means driving maximally right, -pi / 2 maximally left.
     * Value needs to be in (-pi / 2, pi / 2)
     */
    fun setWheelPosition(angle: Float)

    /**
     * Decelerates vehicle speed until 0.
     *
     * @param pressure The percentage of brake pedal use.
     * Number from 0 to 1 where 0 means not using break at all, and 1 means maximal pedal pedal use.
     */
    fun setBreakPedal(pressure: Double)

    /**
     * Accelerates vehicle speed until max speed possible for user.
     *
     * @param pressure The percentage of gas pedal use.
     * Number from 0 to 1 where 0 means not using gas at all, and 1 means maximal gas pedal use.
     */
    fun setGasPedal(pressure: Double)
}
```

이제 모든 차량은 자신들이 어떻게 행동해야 하는지를 설명하는 표준적인 방법이 정해졌다.

대부분의 표준 라이브러리와 인기 있는 라이브러리들은 사용자에게 명확한 기대치와 'contract'를 제공하여, 기능을 안정적으로 사용할 수 있도록 한다.
개발자들 또한 자신의 코드 내에 정의된 요소들에 'contract'를 명확히 하여, 다른 개발자가 해당 요소를 사용할 때, 어떤 행동을 기대할 수 있는지 명확하게 이해할 수 있도록 해야 한다.

## Leaking implementation

구현 세부 사항은 항상 유출된다. 
자동차에서 종류가 다른 엔진은 차를 운전할 수 있지만, 약간 다르게 동작하며, 사용자들은 이러한 차이를 느낄 수 있다.
하지만, 이런 차이점은 'contract'에 명시되지 않았기에 괜찮다.

동일하게, 프로그래밍 언어에서도 구현 세부 사항은 유출된다.  
예를 들어, 리플렉션을 사용하여 함수를 호출하는 것이 가능하지만, 컴파일러에 의해 최적화되지 않는 한 일반 함수를 호출하는 것 보다 훨씬 느리다.
이처럼, 언어가 제시한 'contract'를 지키면서 구현하면 구현 세부 사항의 유출은 큰 문제가 되지 않는다.
이에 따라, 개발자들은 이런 유출이 성능이나 다른 측면에 미칠 수 있는 영향을 이해하고, 좋은 프로그래밍 관행을 적용하여 이러한 문제를 최소화해야 한다.

모든 추상화에서도 구현 세부 사항의 유출을 완전히 막는 것은 불가능하다. 그럼에도 불구하고 최대한 보호해야 한다.
개발자들은 캡슐화를 통해 유출을 보호할 수 있으며, 이는 "내가 허용한 것만 할 수 있고, 그 이상은 할 수 없다"로 설명할 수 있다.
클래스와 함수가 더 많이 캡슐화 될수록, 사용자들이 내부 구현에 의존을 할 수 없기에, 제작자들은 내부 구현에 더 많은 자율성을 가질 수 있다.  