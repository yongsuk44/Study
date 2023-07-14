# Item 33: 생성자 대신 팩토리 함수도 고려하자

Kotlin에서 클래스의 인스턴스를 얻는 가장 일반적인 방법은 기본 생성자(constructor)를 통해 얻는 방법입니다.

```kotlin
class MyLinkedList<T> constructor(val head: T, val tail: MyLinkedList<T>?)

val list = MyLinkedList(1, MyLinkedList(2, null))
```

생성자를 통해 객체의 생성하는 다른 방법이 있습니다.
객체 인스턴스화를 위한 다양한 패턴들이 있는데, 대부분은 직접 객체를 생성하는 대신 함수가 이 역할을 대신 해주는 아이디어를 중심으로 돌아갑니다.

아래는 최상위 함수 `MyLinkedList`의 인스턴스를 생성하는 예시 입니다.

```kotlin
fun <T> myLinkedListOf(vararg element: T): MyLinkedList<T>? {
    if (element.isEmpty()) return null

    val head = element.first()
    val elementTail = element.copyOfRange(1, element.size)
    val tail = myLinkedListOf(*elementTail)
    return MyLinkedList(head, tail)
}

val list = myLinkedListOf(1, 2)
```

위와 같이 생성사 대신 사용되는 함수를 팩토리(Factory) 함수라 하고, 이 함수를 사용하면 다음과 같은 장점들이 있습니다.

- 생성자와 달리 함수는 이름이 존재합니다. 함수의 이름은 객체가 어떻게 생성되고 인수가 무엇인지를 설명할 수 있습니다. 예를 들어, `ArrayList(3)`와 같이 인수가 어떤 의미지 명확하게 이해할 수
  없습니다. 이러한 경우에는 `ArrayList.withSize(3)`와 같이 이름에 혼동을 방지해줄 수 있습니다.
- 함수는 생성자와 달리 반환 유형의 모든 하위 유형의 객체를 반환할 수 있습니다. 이는 각기 다른 객체를 사용해야 하는 경우에 더 나은 객체를 제공하는 데 사용할 수 있습니다.
- 함수는 호출할 때마다 새로운 객체를 생성할 필요가 없습니다. 함수를 사용하여 객체를 생성하면 캐싱 메커니즘을 포함시켜 객체 생성을 최적화하거나 특정 경우에 객체를 재사용할 수 있습니다.
- 객체 외부에 팩토리 함수 정의 시 가시성을 제어하여 같은 파일 이나 같은 모듈에서만 접근할 수 있도록 최상위 팩토리 함수를 생성할 수 있습니다.
- 팩토리 함수는 `inline`와 함께 사용할 경우 타입 파라미터를 실체화(`reified`) 할 수 있습니다.
- 팩토리 함수는 복잡하게 생성될 수 있는 객체를 생성할 수 있습니다.
- 생성자는 상위 클래스의 생성자나 기본 생성자를 즉시 호출해야 하지만, 팩토리 함수 사용 시 생성자 사용을 연기할 수 있습니다.

팩토리 함수는 위와 같은 장점들이 있지만, 기본 생성자와 경쟁하는 것이 아니라는 것을 이해해야 합니다.
팩토리 함수는 본문에서 생성자를 사용해야 하므로 생성자는 반드시 존재해야 합니다.

Kotlin에서는 다양한 종류의 팩토리 함수가 존재합니다.

- Companion object factory function
- Extension factory function
- Top-level factory functions
- Fake constructors
- Methods on a factory class

---

## Companion Object factory function

`companion object`를 통해 팩토리 함수를 정의하는 방법은 다음과 같습니다.

```kotlin
class MyLinkedList<T>(val head: T, val tail: MyLinkedList<T>?) {
    companion object {
        fun <T> of(vararg element: T): MyLinkedList<T>? { /*...*/
        }
    }
}

val list = MyLinkedList.of(1, 2)
```

위와 같은 접근법은 Java의 정적 팩토리 메서드와 동일한 방식이기 떄문에 Java 개발자들에게 매우 친숙한 방법으로 생각 할 수 있습니다.

Kotlin에서, 이러한 접근법은 인터페이스에도 동일하게 동작됩니다.

```kotlin
class MyLinkedList<T>(val head: T, val tail: MyLinkedList<T>?) : MyList<T> {
    // ...
}

interface MyList<T> {
    companion object {
        fun <T> of(vararg element: T): MyList<T>? { /*...*/
        }
    }
}

val list = MyList.of(1, 2)
```

위와 같이 `of` 함수 이름은 따로 설명하지 않아도 대부분의 개발자들이 이해할 수 있어야 합니다.   
그 이유는 Java에서서 같이 `of`라는 이름만으로도 인수가 무엇을 의미하는지 이해하는 것이 충분하다는 관례들이 있기 떄문입니다.

아래는 위와같은 `of`외의 몇 가지 일반적인 이름들입니다.

| 이름                      | 설명                                                                             | 예시                                                               |
|-------------------------|--------------------------------------------------------------------------------|------------------------------------------------------------------|
| from                    | 단일 파라미터를 받고 동일한 타입의 인스턴스를 반환하는 타입 변환 함수입니다.                                    | `val date: Date = Date.from(instant)`                            
| of                      | 다중 파라미터를 받고 그들을 통합하는 동일한 타입의 인스턴스를 반환하는 집합 함수입니다.                              | `val faceCards: Set<Rank> = EnumSet.of(JACK, QUEEN, KING)`       
| valueOf                 | from과 of의 더 자세한 버전입니다.                                                         | `val red: Color = Color.valueOf("RED")`                          
| instance or getInstance | Singleton에서 유일한 인스턴스를 얻는데 사용 <br/> 종종 인수가 동일할 때 항상 반환된 인스턴스가 동일하다고 기대할 수 있습니다. | `val luke: StackWalker = StackWalker.getInstance(options)`       
| create or newInstance   | getInstance와 동일하지만, 이 함수는 각 호출이 새로운 인스턴스를 반환한다는 것을 보장합니다.                      | `val newArray: Array = Array.newInstance(classObject, arrayLen)` 
| getType                 | getInstance와 동일하지만, 이 함수는 팩토리 함수가 다른 클래스에 있을 때 사용합니다.                          | `val fs: FileStore = Files.getFileStore(path)`                   
| newType                 | newInstance와 동일하지만, 이 함수는 팩토리 함수가 다른 클래스에 있을 때 사용합니다.                          | `val br: BufferedReader = Files.newBufferedReader(path)`         |

Kotlin을 사용하는 주니어 개발자들은 Companion Object 멤버들을 단일 블록에 그룹화해야 하는 static 멤버처럼 취급하는 경우가 있습니다.
그러나, Companion Object는 실제로 훨씬 더 강력합니다. 예를 들어, Companion Object는 인터페이스를 구현하고 클래스를 확장할 수 있습니다.

---

## Extension factory functions

가끔씩, 기존의 `companion object` 함수처럼 작동하는 팩토리 함수를 생성하고자 하는 경우가 있습니다.
그런데 이 `companion object`를 수정할 수 없거나, 혹은 별도의 파일에 새 함수를 명시하고 싶은 경우가 있을 수 있습니다.

이럴 때, `companion object`의 또 다른 장점인 확장 함수를 정의할 수 있습니다.

예를 들어, `Tool` 인터페이스를 변경할 수 없다고 가정하고 아래 코드를 보시죠

```kotlin
interface Tool {
    companion object { /*...*/ }
}
```

그럼에도 여전히 아래와 같이 `companion object`에 대한 확장 함수를 정의할 수 있습니다.

```kotlin
fun Tool.Companion.createBigTool(/*...*/): BigTool = { /*...*/ } 
```

그러면 이제, `Tool.createBigTool()`를 호출하여 팩토리 함수를 사용할 수 있습니다.

위와 같은 방식은 외부 라이브러리를 팩토리 메서드로 확장하는데 매우 강력한 방법입니다.   
단지 주의해야 할 점은, `companion object`를 통한 확장 함수를 만들기 위해서는 객체가 비어 있더라도 어떠한 형태의 `companion object`가 필요합니다.

```kotlin
interface Tool {
    companion object {}
}
```

---

## Top-level functions

객체 생성 시 가장 많이 보편화된 방법은 최상위 팩토리 함수를 사용하는 것입니다. 대표적으로 `listOf`, `setOf`, `mapOf` 등이 있습니다.

예를 들어 안드로이드에서는 `Activity`를 시작하기 위한 `Intent`를 생성하는 함수를 정의하는 방법이 있습니다.

```kotlin
class MainActivity : AppCompatActivity() {
    companion object {
        fun getIntent(context: Context): Intent = Intent(context, MainActivity::class.java)
    }
}
```

Kotlin 'Anko' 라이브러리에서는 이와 같이 `reified type`을 가진 최상위 함수인 `intentFor`를 사용할 수 있습니다. `intentFor<MainActivity>()`

또한, 이 함수는 인자를 넘겨주는 것도 가능합니다. `intentFor<MainActivity>("page to 2", "row to 5")`

`List`나 `Map` 같은 경우에는 최상위 팩토리 함수를 이용한 객체 생성이 굉장히 효과적입니다.
왜냐하면 `listOf(1,2,3)`는 `List.of(1,2,3)`보다 훨씬 간결하고 읽기 쉽기 때문입니다.

하지만, `public` 최상위 함수를 사용할 때는 주의해야 합니다. `public` 최상위 함수의 단점은 어디에서나 접근이 가능하다는 것이기 때문에, IDE의 팁들을 혼란스럽게 만들 수 있습니다.
이 문제는 최상위 팩토리 함수가 클래스 메소드와 같은 이름을 가질 때 문제가 될 수 있습니다. 따라서, 최상위 팩토리 함수의 이름 작명 시 주의해야 합니다.

---

## Fake constructors

Kotlin에서는 생성자를 최상위 함수처럼 사용할 수 있습니다.

```kotlin
class A
val a = A()
```

그리고 `val refrence: () -> A = ::A`처럼 최상위 함수와 같이 참조가 됩니다. (생성자 참조는 함수 인터페이스 구현)

이러한 코드는 생성자와 함수 사이에서 차이점으로 대소문자의 차이 밖에 보이지 않습니다.
하지만, 일반적으로 클래스는 대문자로 시작하고, 함수는 소문자로 시작합니다.

함수가 대문자로 시작하는 경우도 있긴 합니다. Kotlin 표준 라이브러리의 경우 `List`와 `MutableList`는 인터페이스 형태로 생성자를 가질 수 없습니다.
그럼에도 불구하고, Kotlin 개발자는 다음과 같이 `List(4) { "Count : $it" }` 생성의 요구가 있었고 이는 Kotlin 1.1부터 지원하게 되었습니다.

```kotlin
public inline fun <T> List(
    size: Int,
    init: (index: Int) -> T
): List<T> = MutableList(size, init)

public inline fun <T> MutableList(
    size: Int,
    init: (index: Int) -> T
): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { index -> list.add(init(index)) }
    return list
}
```

이와 같은 최상위 함수들은 생성자처럼 보이고 작동함과 동시에 팩토리 함수의 모든 장점을 가지고 있습니다.   
개발자들은 이들이 내부적으로 최상위 함수라는 것을 알지 못하기에 종종 **Fake Constructors**라고 부릅니다.

개발자들이 가짜 생성자(Fake Constructors)를 선택하는 주된 이유는 다음과 같습니다.

- 인터페이스에 `constructor`가 있어야 하는 경우
- `reified type argument`가 필요한 경우

그 외에는 가짜 생성자는 일반 생성자처럼 보이고 작동해야 합니다. 
만약에 캐싱을 포함하고 싶거나, `nullable` 타입을 반환하거나, 생성 가능한 클래스의 서브클래스를 반환하려는 경우 
`companion object` 팩토리 메소드와 같이 이름이 있는 팩토리 함수를 사용해야 합니다.

가짜 생성자를 선언하는 또 다른 방법도 있습니다. `companion object`에 `invoke` 연산자를 사용하면 비슷한 결과를 얻을 수 있습니다.

```kotlin
class Tree<T> {
    companion object {
        operator fun <T> invoke(size: Int, generator: (Int) -> T): Tree<T> {
            // ...
        }
    }
}

Tree(10) { "$it" }
```

그러나 `companion object`에서 `invoke`를 구현하여 가짜 생성자를 만드는 방법은 아래와 같은 두 가지 주요 이유로 인해 권장되지 않습니다.

`invoke` 라는 이름은 객체를 '호출' 하는 연산을 의미합니다. 이것은 객체를 '생성'하는 연산과는 별개의 개념입니다.
따라서 이 방식으로 가짜 생성자를 만드는 것은 용어의 일관성을 해칩니다.

또한 `companion object`의 `invoke` 연산자를 사용하여 가짜 생성자를 만드는 것은 일반적인 최상위 함수를 사용하는 것보다 복잡합니다.
이 복잡성은 리플렉션(Reflection)을 사용하여 코드를 조사할 때 특히 드러납니다.
리플렉션은 프로그램이 실행 중 자신의 구조를 조사하고 수정할 수 있는 능력을 가진 Java의 특징 입니다.
리플렉션을 사용하여 `Constructor`, `Fake constructor`, `companion object invoke()`를 참조할 때의 코드를 비교해보면,
`companion object invoke()`는 다른 2가지 방식보다 더 복잡한 형태를 띄우고 있기에 이 방식은 권장되지 않습니다.

Constructor: `val f: () -> Tree = ::Tree`  
Fake constructor: `val f: () -> Tree = ::Tree`  
Invoke in Companion object : `val f: () -> Tree = Tree.Companion::invoke`

따라서 가짜 생성자가 필요한 경우 일반적인 최상위 함수를 사용하는 것이 좋습니다.
이것은 가짜 생성자가 클래스 자체에서 생성자를 정의할 수 없거나, 생성자가 제공하지 않는 기능(==refied type parameter)이 필요할 때 사용될 수 있습니다.