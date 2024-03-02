Kotlin에서 클라이언트에게 클래스 인스턴스를 제공하는 가장 일반적인 방법은 'primary constructor'를 제공하는 것이다.

```kotlin
class MyLinkedList<T>(val head: T, val tail: MyLinkedList<T>?)

val list = MyLinkedList(head = 1, tail = MyLinkedList(head = 2, tail = null))
```

객체 생성에 있어 생성자만이 유일한 수단이 아니며, 객체의 인스턴스화를 위한 다양한 'creational 패턴'이 존재한다.  
대부분의 'creational 패턴'은 직접 객체를 생성하는 대신, 함수가 이 역할을 대신하는 아이디어를 중심으로 돌아간다.

다음 'Top-level 함수'는 `MyLinkedList`의 인스턴스를 생성하는 예시이다.

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

이처럼 생성자의 역할을 대신하는 함수를 'factory function'이라 하며, 생성자 대신 팩토리 함수를 사용하면 다음과 같은 장점들이 있다.

생성자와 달리 함수에는 이름이 존재하며, 함수의 이름은 객체가 어떻게 생성되고 인자가 무엇인지를 설명해준다.
예를 들어, 'ArrayList(3)'과 같이 인자가 무엇을 의미하는지 명확하게 설명할 수 없다.
이러한 상황에서 'ArrayList.withSize(3)'와 같이 팩토리 함수를 사용하면 혼동을 방지할 수 있다.
즉, 팩토리 함수는 이름을 통해서 객체의 생성 과정이나 특성을 명확히 할 수 있으며, 동일한 타입의 파라미터를 가진 생성자 간의 충돌을 해결할 수 있다.

생성자와 달리 함수는 반환 타입의 어떠한 하위 타입도 반환할 수 있기에 다양한 상황에서 더 적절한 객체를 제공할 수 있다.
이는 실제 객체 구현을 인터페이스 뒤에 숨기고자 할 때 중요하다. 표준 라이브러리의 'listOf()'의 반환 타입은 'List' 인터페이스이다.
실제로 'List' 인터페이스를 통해 반환되는 객체는 플랫폼에 따라 다르며, Kotlin/JVM, Kotlin/JS, Kotlin/Native에서는 각기 다른 내장 컬렉션을 사용한다.
또한, 시간이 지나면서 'List'의 실제 타입이 변경될 수 있고, 새로운 객체가 여전히 'List' 인터페이스를 구현하고 동일한 방식으로 동작한다면 자연스럽게 사용될 수 있기에 많은 유연성을 제공한다.

생성자와 달리 팩토리 함수는 호출될 때마다 새로운 객체를 생성할 필요가 없다.
이는 팩토리 함수를 통해 객체 생성 과정에 'singleton 패턴'과 같은 캐싱 메커니즘을 적용하거나, 특정 조건 하에서만 객체를 생성하는 등의 최적화를 할 수 있기에 유용하다.
또한, 객체 생성 과정에서의 실패를 안전하게 처리할 수 있도록 'null'을 반환하는 정적 팩토리 함수를 정의할 수 있다.
예를 들어, 어떤 이유로 'Connection'을 생성할 수 없을 때 'null'을 반환하는 'Connections.createOrNull()'를 정의할 수 있다.

팩토리 함수는 아직 존재하지 않는 객체를 제공할 수 있다.
이는 DI 라이브러리인 'Dagger'와 같이 'annotation processing'을 기반으로 하는 라이브러리 제작자들에 의해 집중적으로 사용된다.
이 방식을 통해 프로그래머들은 프로젝트를 빌드하지 않고도 생성될 객체나 프록시를 통해 사용될 객체에 대해 작업할 수 있다.

객체 외부에 팩토리 함수를 정의할 때, 'visibility'를 제어할 수 있다.
예를 들어, 'Top-level 팩토리 함수'를 동일한 파일이나 모듈 내에서만 접근할 수 있도록 제한할 수 있다.

팩토리 함수는 'inline'으로 선언될 수 있으며, 이로 인해 팩토리 함수의 타입 파라미터를 '실체화(reified)' 할 수 있다.
이를 통해, 런타임에 제네릭 타입 정보가 지워지는 'type erasure' 문제를 해결할 수 있다.
즉, 'reified 타입 파라미터'를 사용하면, 런타임에도 타입 정보를 유지하고 접근할 수 있어, 타입 검사나 캐스팅과 같은 작업을 보다 안전하게 수행할 수 있다.

팩토리 함수는 생성자로 구성하기 복잡한 객체를 생성할 수 있다.
예를 들어, 여러 단계의 초기화가 필요하거나, 다양한 파라미터와 의존성을 기반으로 객체를 구성해야 하는 등의 복잡성을 내부적으로 캡슐화하고 사용자에게는 단순한 인터페이스를 제공할 수 있다.

생성자는 상위 클래스의 생성자나 'primary constructor'를 즉시 호출해야 하지만, 팩토리 함수를 사용할 때는 생성자 사용을 연기할 수 있다.
예를 들어, 초기화에 필요한 데이터를 비동기로 불러와야 하거나, 특정 조건에 의해 객체 생성을 결정해야 하는 경우에 생성자 호출을 지연시키거나 조건적으로 실행할 수 있다.

```kotlin
fun makeListView(userRepository: UserRepository): ListView {
    val users = userRepository.fetchUserInfo()      // Here we read items from repository
    return ListView(items)                          // We call actual constructor
}
```

팩토리 함수 사용에는 한 가지 제한이 있다.
상속 구조에서 하위 클래스의 생성은 상위 클래스의 생성자를 통해 초기화하는 과정을 반드시 거쳐야 하기에, 팩토리 함수는 하위 클래스의 생성자에서 직접적으로 호출할 수 없다.

```kotlin
class IntLinkedList : MyLinkedList<Int>() {      // Supposing that MyLinkedList is open

    constructor(vararg ints: Int) : myLinkedListOf(*ints)    // Error
}
```

하지만, 이는 문제가 되지 않는다.
왜냐하면, 팩토리 함수를 통해 상위 클래스를 생성하기로 결정했다면, 당연하게도 하위 클래스를 위한 팩토리 함수도 정의하면 되기 때문이다.

```kotlin
class MyLinkedIntList(head: Int, tail: MyLinkedIntList?) : MyLinkedList<Int>(head, tail)

fun myLinkedIntListOf(vararg elements: Int): MyLinkedIntList? {
    if (elements.isEmpty()) return null
    val head = elements.first()
    val tail = myLinkedIntListOf(*elements.copyOfRange(1, elements.size))
    return MyLinkedIntList(head, tail)
}
```

위 'myLinkedIntListOf' 팩토리 함수는 이전에 정의한 생성자보다 길지만, 유연성과 클래스의 독립성 및 Nullable 반환 타입을 선언할 수 있는 능력 등 더 나은 특성을 가지고 있다.

팩토리 함수에는 위에서 설명한 것과 같은 많은 장점들이 있지만, 이해해야 할 점은 팩토리 함수가 'primary constructor'와 경쟁하는 것이 아니라는 것이다.
팩토리 함수도 본문의 생성자를 사용해야 하므로, 생성자는 반드시 존재해야 한다. 거의 사용되지 않지만, 객체 생성을 팩토리 함수로 강제하고 싶다면 생성자를 'private'으로 선언 할 수 있다.

팩토리 함수는 주로 'secondary constructor'와 경쟁하지만, 대부분의 프로젝트에서는 'secondary constructor'를 거의 사용하지 않기에 팩토리 함수가 경쟁에 우위를 가지고 있다.
또한, 팩토리 함수는 다양한 종류가 있기에 스스로와도 경쟁 상대가 된다.

아래는 다양한 팩토리 함수들의 종류이다.

- Companion object factory function
- Extension factory function
- Top-level factory functions
- Fake constructors
- Methods on a factory class

## Companion Object Factory Function

팩토리 함수를 정의하는 가장 일반적인 방법은 'companion object' 안에 정의하는 것이다.

```kotlin
class MyLinkedList<T>(val head: T, val tail: MyLinkedList<T>?) {

    companion object {
        fun <T> of(vararg elements: T): MyLinkedList<T>? {
            // ... 
        }
    }
}

// usage
val list = MyLinkedList.of(1, 2)
```

이러한 접근 방식은 'static factory method'와 직접적으로 동일하기에 Java 개발자에게는 매우 친숙할 것이다.
또한, 다른 언어의 개발자도 익숙할 수 있는데, C++과 같은 일부 언어에서는 생성자와 사용법이 비슷하지만, 이름이 있다는 점 떄문에 'Named Constructor Idiom'이라고 불린다.

Kotlin에서는 이러한 접근 방식이 인터페이스에서도 동작한다.

```kotlin
class MyLinkedList<T>(val head: T, val tail: MyLinkedList<T>?) : MyList<T> {
    // ...
}

interface MyList<T> {
    // ...

    companion object {
        fun <T> of(vararg elements: T): MyList<T>? {
            // ...
        }
    }
}

val list = MyList.of(1, 2)
```

'of' 함수의 이름이 서술적이지 않음에도 불구하고 대부분의 개발자들이 이해할 수 있다는 점을 주목해야 한다.
이는 Java에서 유래한 몇 가지 'convention'이 있고, 이 덕분에 'of'와 같은 짧은 단어만으로도 인자가 무엇을 의미하는지 이해할 수 있다.

아래는 몇 가지 일반적인 이름과 그에 대한 설명이다.

| 함수명                     | 설명                                                                                                                                                                                               |
|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| from                    | - 단일 타입의 특정 인스턴스를 다른 타입의 인스턴스로 변환하는데 사용된다. <br/> - `val date: Date = Date.from(instant)`                                                                                                         |
| of                      | - 동일한 타입의 여러 인스턴스를 결합하여 하나의 컬렉션으로 집계하는데 사용된다. <br/> - `val faceCards: Set<Rank> = EnumSet.of(JACK, QUEEN, KING)`                                                                                 |
| valueOf                 | - 기본 타입의 값을 받아 해당 값을 갖는 래퍼 클래스의 인스턴스로 변환하는데 사용된다. <br/> - `val prime: BigInteger = BigInteger.valueOf(Integer.MAX_VALUE)`                                                                        |
| instance or getInstance | - 'singleton 패턴'에서 유일한 인스턴스를 얻는데 사용된다. <br/> - 파라미터화가 된 경우, 제공된 인자에 따라 파라미터화된 인스턴스를 반환하며, 인자가 같을때 반환되는 인스턴스가 항상 같다고 예상할 수 있다. <br/> - `val luke: StackWalker = StackWalker.getInstance(options)` |
| create or newInstance   | - 호출 할 때 마다 새로운 인스턴스를 생성하며, 이를 반환한다. <br/>- `val newArray: Array = Array.newInstance(classObject, arrayLen)`                                                                                     |
| getType                 | - 유일한 인스턴스를 얻는 팩토리 함수가 클래스 내부에 존재하지 않고, 다른 클래스에 정의되어 있을 때 사용된다. <br/> - `val fs: FileStore = Files.getFileStore(path)`                                                                           |
| newType                 | - 새로운 인스턴스를 얻는 팩토리 함수가 클래스 내부에 존재하지 않고, 다른 클래스에 정의되어 있을 떄 사용된다. <br/> - `val br: BufferedReader = Files.newBufferedReader(path)`                                                                 |

Kotlin 경험이 부족한 개발자들은 'companion object 멤버'를 단일 블록에 그룹화해야 하는 'static 멤버'처럼 취급하는 경우가 있다.
하지만, 실제로 'companion object'는 훨씬 더 강력한 기능을 가지고 있다.
클래스 내부에 'companion object'가 있으면, 해당 클래스의 인스턴스 없이도 접근할 수 있는 멤버를 제공한다.
또한, 인터페이스를 구현하거나 다른 클래스를 상속할 수 있기에, 싱글톤 또는 다형성을 구현할 수 있다.

이런 기능을 통해, 아래와 같은 'companion object 팩토리 함수'를 구현할 수 있다.

```kotlin
abstract class ActivityFactory {
    abstract fun getIntent(context: Context): Intent

    fun start(context: Context) {
        val intent = getIntent(context)
        context.startActivity(intent)
    }

    fun startForResult(activity: Activity, requestCode: Int) {
        val intent = getIntent(activity)
        activity.startActivityForResult(intent, requestCode)
    }
}

class MainActivity : AppCompatActivity() {
    // ...

    companion object : ActivityFactory() {
        override fun getIntent(context: Context): Intent =
            Intent(context, MainActivity::class.java)
    }
}

// Usage
val intent = MainActivity.getIntent(context)
MainActivity.start(context)
MainActivity.startForResult(activity, requestCode)
```

이처럼 'abstract companion object factory'은 단순히 메서드만 가질 수 있는 것이 아니라, 데이터를 저장하고 이 데이터를 기반으로 로직을 수행할 수 있다.
이를 통해, 캐싱을 위한 데이터 구조를 보유하거나, 테스트로 사용될 Fake 객체 생성에 필요한 데이터를 제공할 수 있다.

예를 들어, Coroutines 라이브러리의 대부분 'companion object'가 'CoroutineContext.Key' 인터페이스를 구현하고 있으며,
이는 모든 'CoroutineContext'를 식별하는 키로 사용된다.

![img.png](companion_object_factory.png)

---

## Extension factory functions

때때로 기존의 'companion object'를 함수처럼 동작하는 팩토리 함수로 생성하고 싶을 때가 있을 수 있다.  
이 때 만약, 'companion object'를 수정할 수 없거나, 별도의 파일에 새 함수를 명시하고 싶은 경우에 확장 함수로 정의할 수 있다.

아래와 같이 'Tool' 인터페이스를 변경할 수 없다고 가정해보자.

```kotlin
interface Tool {
    companion object { /*...*/ }
}
```

그럼에도 불구하고, 클래스나 인터페이스에 'companion object'가 선언되어 있을 때, 아래와 같이 확장 함수를 정의할 수 있다.

```kotlin
fun Tool.Companion.createBigTool(/*...*/): BigTool = { /*...*/ } 
```

그런 다음, `Tool.createBigTool()`과 같이 호출할 수 있다.

이런 방식은 외부 라이브러리를 팩토리 함수로 확장할 수 있게하는 강력한 방법으로, 추가적인 생성자 또는 팩토리 함수를 제공할 수 있다.
하지만, 'companion object'의 확장을 만들기 위해서는 비어 있더라도 반드시 `companion object`가 클래스 내에 선언되어 있어야 한다는 전제 조건이 있다.

```kotlin
interface Tool {
    companion object {}
}
```

---

## Top-level functions

객체를 생성하는 인기 있는 방법 중 하나는 'listOf', 'setOf', 'mapOf'와 같은 Top-level 팩토리 함수를 사용하는 것이다.
마찬가지로, 라이브러리 제작자들도 객체를 생성하기 위해 Top-level 함수를 지정한다.
이처럼 Top-level 팩토리 함수는 널리 사용되며, 예를 들어 안드로이드에서는 'Activity'를 시작하기 위한 'Intent'를 생성하는 함수를 정의하는 전통적인 방법이 있으며,
다음과 같이 'companion object' 함수로 'getIntent()'를 작성할 수 있다.

```kotlin
class MainActivity : AppCompatActivity() {
    companion object {
        fun getIntent(context: Context): Intent =
            Intent(context, MainActivity::class.java)
    }
}
```

위 코드를 'Anko' 라이브러리에서는 'reified' 타입을 사용하여 다음과 같이 작성할 수 있다. `intentFor<MainActivity>()`

또한, 이 함수는 인자를 넘겨주는 것도 가능하다. `intentFor<MainActivity>("page to 2", "row to 5")`

'List'나 'Map'과 같이 자주 생성되는 객체에 대해서 Top-level 함수를 사용한 객체 생성은 최고의 선택일 것이다.
예를 들어, 'listOf(1, 2, 3)'은 'List.of(1, 2, 3)' 보다 더 단순하고 읽기 쉽다.
하지만, public top-level 함수는 어디에서나 사용할 수 있는 단점이 있어 신중하게 사용되어야 한다.
특히, public top-level 함수는 클래스 메서드처럼 명명되어 혼동을 일으킬 때 문제가 될 수 있다.
이 때문에 public Top-level 함수의 이름을 지을 때는 현명하게 지어야 한다.

---

## Fake constructors

Kotlin에서 생성자는 Top-level 함수와 같은 방식으로 사용된다.

```kotlin
class A

val a = A()
```

또한, 생성자 참조는 Top-level 함수와 동일하게 참조된다. (생성자 참조는 함수 인터페이스를 구현한다.)

```kotlin
val reference: () -> A = ::A
``` 

사용 측면에서 볼 때, 생성자와 함수 사이의 유일한 차이점은 관례적으로 클래스는 대문자로 시작하고, 함수는 소문자로 시작한다는 점이다.
하지만, 기술적으로 함수가 대문자로 시작될 수 있다. 예를 들어, Kotlin 표준 라이브러리의 경우 'List'와 'MutableList' 인터페이스가 여기에 해당된다.  
이 인터페이스들은 생성자를 가질 수 없지만, Kotlin 개발자들은 아래와 같이 'List'를 생성하고 싶어하는 니즈가 있었다.

```kotlin
List(4) { "User$it" } // [User0, User1, User2, User3]
```

그렇기에 'Kotlin 1.1'부터 'Collections.kt'에 아래와 같은 함수가 포함되었다.

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

위와 같은 Top-level 함수들은 생성자처럼 보이고 동작하지만, 팩토리 함수의 모든 장점을 가지고 있다.  
하지만, 많은 개발자들이 이들이 실제로는 Top-level 함수라는 사실을 모른다. 이러한 이유로 종종 Fake 생성자라고 불린다.

개발자들이 실제 생성자보다 Fake 생성자를 선택하는 주된 이유는 다음과 같다.

- 인터페이스에 대한 생성자를 가지기 위해
- 구체화된 타입 인자를 가지기 위해

그 외에 Fake 생성자는 일반 생성자처럼 보이고 동작해야 한다.
만약, 캐싱을 포함하거나, nullable 타입을 반환하거나, 생성될 수 있는 클래스의 서브 클래스를 반환하고 싶다면,
이름을 가진 팩토리 함수(e.g : companion object 팩토리 함수)를 사용하는 것이 더 적절할 수 있다.

Fake 생성자를 선언하는 또 다른 방법은 'companion object'와 'invoke' 연산자를 사용하여 비슷한 결과를 얻을 수 있다.

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

그러나, 'companion object'에서 'invoke'를 구현하여 Fake 생성자를 만드는 것은 
'companion object'를 'invoke' 하는 것이 어떤 의미인지 명확하지 않기 때문에 권장하지 않는다. ('Item 12' 위반)

```kotlin
Tree.invoke(10) { "$it" }
```

'Invocation'은 객체 생성과는 다른 작업이다. 연산자를 이런 식으로 사용하는 것은 그 이름과 일치하지 않는다.
더 중요한 점은 이런 접근 방식이 단순히 Top-level 함수를 사용하는 것보다 더 복잡하다는 점이다.
생성자, Fake 생성자, 그리고 companion object의 invoke 함수를 참조할 때 리플렉션이 어떻게 차이나는지 비교해보면 이 복잡성을 알 수 있다.

- Constructor: `val f: () -> Tree = ::Tree`
- Fake constructor: `val f: () -> Tree = ::Tree`
- Invoke in companion object: `val f: () -> Tree = Tree.Companion::invoke`

이처럼 Fake 생성자가 필요할 때는 일반적인 Top-level 함수를 사용하는 것이 좋으며 다음과 같은 경우에 조심스럽게 사용해야 한다.
- 클래스 자체에 생성자를 정의할 수 없을 때
- 생성자와 유사한 용도로 제공하지만 생성자가 제공하지 않는 기능(e.g : reified type parameter)이 필요할 때

## Methods on a factory class

팩토리 클래스와 관련된 많은 'creational 패턴'이 있다.   
예를 들어, 'abstract 팩토리' 또는 'prototype' 등이 있으며, 각각의 패턴은 나름의 장점을 가진다.
이런 접근 방법 중 'telescoping constructor'와 'builder pattern'과 같이 일부는 Kotlin에서 적합하지 않다는 것을 알 수 있다.

팩토리 클래스는 클래스가 상태를 가질 수 있기에 팩토리 함수보다 장점을 가진다.  
예를 들어, 다음 ID 번호를 갖는 학생들을 생성하는 아주 간단한 팩토리 클래스입니다.

```kotlin
data class Student(
    val id: Int,
    val name: String,
    val surname: String
)

class StudentsFactory {
    var nextId = 0
    fun next(name: String, surname: String) = 
        Student(nextId++, name, surname)
}

val factory = StudentsFactory()
val student = factory.next("Alice", "Smith")
println(student)        // Student(id=0, name=Alice, surname=Smith)

val anotherStudent = factory.next("Bob", "Johnson")
println(anotherStudent) // Student(id=1, name=Bob, surname=Johnson)
```

팩토리 클래스가 프로퍼티를 가질 수 있다는 것은 객체 생성 과정을 보다 최적화하는데 사용될 수 있음을 말한다.
상태를 유지하는 능력 덕분에, 개발자는 다양한 종류의 최적화나 기능을 도입할 수 있다.
에를 들어, 캐싱을 사용하거나 이전에 생성된 객체를 복제하여 객체 생성 속도를 높일 수 있다.