# Generics: `in`, `out`, `where`

Kotlin의 클래스는 Java와 마찬가지로 타입 파라미터를 가질 수 있으며, 해당 클래스의 인스턴스를 만드려면 타입 인자를 제공하면 됩니다.  
하지만 파라미터가 생성자에서 타입이 추론이 된다면 타입 파라미터를 생략할 수 있습니다.

```kotlin
class Box<T>(t: T) {
    var value = t
}

val box: Box<Int> = Box(1)
val box1 = Box(1)
```

---

## Variance

> Java의 타입 시스템은 기본적으로 공변성을 지원하지 않기에 직접 와일드 카드를 구현 해줘야함,  
> 그러나 Kotlin은 이런 공변성을 `out`, `in` 키워드로 지원하기에 더 간단하게 구현할 수 있음
>
> - `out`(공변) : 선언된 타입 또는 그 하위 타입의 객체를 읽을 수 있지만, 새롭게 정의할 수 없음 -- '읽기 전용' 작업
> - `in`(반공변) : 선언된 타입 또는 그 상위 타입의 객체를 새로이 정의할 수 있지만, 타입을 정확히 알지 못해 읽을 수 없음 -- '쓰기 전용' 작업

Java의 제네릭은 클래스, 인터페이스, 함수 등에서 동일한 코드를 재사용하고 싶을 때 여러 타입을 지원하는 중요한 기능 중 하나입니다.  
이처럼 중요한 기능 중 하나인 제네릭 타입을 사용할 때, Java 타입 시스템은 와일드카드 타입을 필요로 합니다.  
(Kotlin은 와일드카드 대신 [declaration-site variance](#declaration-site-variance)와 [Type projection](#type-projections)가 존재합니다.)

자세히 알아보면 Java의 제네릭 타입은 불변(invariance)입니다.  
타입 불변성은 제네릭 타입을 사용하는 클래스, 인터페이스에는 해당 타입의 상위, 하위를 대입할 수 없고 오직 일치하는 타입만을 대입하는 것을 의미합니다.

이처럼 Java의 `List<T>`를 활용한 `List<String>`와 `List<Object>`는 서로 다른 타입으로 취급됩니다.  
만약 `List<String>`과 `List<Object>`가 같은 타입으로 취급된다면, 런타임에서 문제가 발생할 수 있기 때문입니다.

```java
List<String> strs=new ArrayList<String>();
        List<Object> objs=strs; // A compile-time error here saves us from runtime exception later.
        objs.add(1); // Put an Integer into a list of Strings
        String s=strs.get(0); // ClassCastException: Cannot cast Integer to String
```

위 상황와 같이 Java는 런타임 안전성을 확보하기 위해 일부 타입 관련 작업을 제한합니다.  
이로 인해 `Collection`의 `addAll()`과 같은 메서드의 시그니처가 복잡해질 수 있습니다.

`addAll()`을 직관적으로 작성하면 다음과 같을 수 있을겁니다.

```java
interface Collection<E> {
    void addAll(Collection<E> items);
}
```

그러나 위처럼 작성되면, 아래와 같은 작업을 할 수 없게 됩니다.

```java
void copyAll(Collection<Object> to,Collection<String> from){
        to.addAll(from);
        }
```

왜냐하면 `Collection<String>`은 `Collection<Object>`의 하위 타입임을 확인할 수 없기 때문입니다.  
그래서 `addAll()`의 실제 시그니처는 다음과 같습니다.

```java
interface Collection<E> {
    void addAll(Collection<? extends E> items);
}
```

`? extends E` 타입의 와일드카드는 Java에서 공변성(covariance)을 구현하게 도와줍니다.

`? extends E` 와일드카드는 `E`와 `E`의 하위 타입들을 안전하게 읽을 수 있게(read) 해주지만,   
알려지지 않은 `E`의 하위 타입에 어떤 객체가 부합하는지 알 수 없기에 쓸 수는(write) 없게 만듭니다.

`Collection<String>`은 `Collection<? extends Object>`의 하위 타입으로 취급되기 때문에,  
[상한](#상한)을 가진 와일드카드는 타입을 공변(covariance)으로 만듭니다.

반대로 컬렉션에 항목을 추가만 할 수 있다면, `Collection<? super String>`로 선언하여 `String`또는 그 상위 타입만을 수용할 수 있도록 할 수 있습니다.

`Collection<? super String>`와 같은 와일드카드를 반공변(contravariance)이라고 부릅니다.  
이 와일드카드는 `add(String)`, `set(int, String)`을 호출할 수 있습니다.

그러나 `List<T>`에서 `T`를 반환을 호출하면 `String`이 아닌, `Object`를 얻게 됩니다.

---

### Declaration-site variance

Java에서 오직 `T`를 반환하는 메서드만 있는 제네릭 인터페이스 `Source<T>`가 있다고 가정해보면 다음과 같을 것입니다.

```java
interface Source<T> {
    T nextT();
}
```

여기서 `Source<T>`가 `T`를 파라미터로 취급하는 메서드가 없고, 오직 `T`를 반환하는 메서드만 존재하기에  
`Source<String>`의 인스턴스에 대한 참조를 `Source<Object>` 타입의 변수에 저장하는 것은 안전할 수 있습니다.

그러나 **Java의 타입 시스템은 기본적으로 공변성을 지원하지 않기에** 아래 구현은 금지하고 있습니다.

```java
void demo(Source<String> strs){
        Source<Object> objects=strs; // !!! Not allowed in Java
        }
```

이러한 제약을 해결하기 위해 `Source<? extends Object>`와 같은 타입을 사용하면 컴파일러의 타입 안전성 체크를 통과할 수 있을겁니다.

Kotlin에서는 이런 종류의 문제를 컴파일러에게 간단히 설명할 수 있는 방법이 있으며 이를 **Declaration-site variance**라고 합니다.

---

### out

`Source`의 타입 파라미터 `T`에 `out` 수정자를 달아 `Source<T>`의 멤버들을 오직 생산만 하고 소비되지 않도록 할 수 있습니다.

```kotlin
interface Source<out T> {
    fun nextT(): T
}

fun demo(strs: Source<String>) {
    val objects: Source<Any> = strs // This is OK, since T is an out-parameter
}
```

`out`은 공변성을 나타내는 키워드로 타입 파라미터가 생산자로만 동작하고, 소비자로 동작하지 않음을 의미합니다.

이로 인해, 상속 관계에 있는 타입 간에도 안전하게 할당이 가능해집니다.  
예를 들어, `class C<out T>`으로 선언되면 `C<Drived>`의 인스턴스를 `C<Base>`타입에 안전하게 할당할 수 있습니다.

이처럼 공변을 적용할 때 Kotlin에서의 **Declaration-site variance**는 Java의 와일드카드 보다 더 간단하고 직관적입니다.

### in

Kotlin에서는 공변성을 정의하는 `in` 수정자도 지원합니다.   
`in`은 타입 파라미터가 오직 소비되기만 하고 생산되지 않음을 의미합니다.

반공변 타입의 좋은 예시는 아래 `Comparable` 입니다.

```kotlin
interface Comparable<in T> {
    operator fun compareTo(other: T): Int
}

fun demo(x: Comparable<Number>) {
    x.compareTo(1.0) // 1.0 has type Double, which is a subtype of Number
    // Thus, we can assign x to a variable of type Comparable<Double>
    val y: Comparable<Double> = x // OK!
}
```

---

## Type projections

> - Type projections : 클래스, 인터페이스에서 `in`과 `out`을 통해 타입 파라미터의 사용을 제한하는 것
> - User-site variance : 클래스, 인터페이스를 정의할 때가 아닌, 실제로 그 타입을 사용하는 코드에서 제네릭 타입의 공변성 or 반공변성을 지정
> - Star-projections `<*>` : 제네릭 타입을 모를 때 해당 타입을 안전하게 사용할 수 있도록 하는 문법

### Use-site variance: type projections

클래스와 인터페이스를 정의할 때 타입 파라미터 `T`를 `out`으로 선언하여 사용지점에서 하위 타입 문제를 피할 수 있지만,   
`Array`와 같은 클래스들은 타입 파라미터 `T`에 대해 공변성(`out`)과 반공변성(`in`)을 둘 다 가질 수 없습니다.

```kotlin
class Array<T>(vak size: Int) {
    operator fun get(index: Int): T {
        ...
    }
    operator fun set(index: Int, value: T) {
        ...
    }
}
```

이처럼 공변성과 반공변성을 가질 수 없다는 점은 다음과 같은 유연성을 제한합니다.

아래 `copy()`는 하나의 배열에서 다른 배열로 아이템을 복사하는 것을 목표로 합니다.

```kotlin
fun copy(from: Array<Any>, to: Array<Any>) {
    assert(from.size == to.size)

    for (i in from.indices) {
        val fromValue = from.get(index = i)
        to.set(index = i, value = fromValue)
    }
}

val ints: Array<Int> = arrayOf(1, 2, 3)
val any = Array<Any>(3) { "" }
copy(from = ints, to = any) // type is Array<Int> but Array<Any> was expected
```

위와 같이 `Int` → `Any` 타입으로 복사할 때 제네릭 타입의 불변성에 문제가 발생됩니다.

`Array<T>`는 `T`에 대해 불변성이 있어서 서로 다른 타입의 배열 간에는 호환성을 가질 수 없습니다.  
즉, `Array<Int>`와 `Array<Any>`는 서로 호환되지 않습니다.

이러한 문제를 해결하기 위해 다음과 같이 `copy()`를 변경할 수 있습니다.

```kotlin
fun copy(from: Array<out Any>, to: Array<Any>) {
    ...
}
```

이것을 **Type projection**(타입 투영)이라 부릅니다.

여기서 `from`은 단순한 배열이 아니라 제한된(projections) 배열을 의미합니다.  
이는 `copy()`의 `from` 파라미터는 `Array<T>`에서 **타입 파라미터 `T`와 그 하위 타입을 반환**하는 메서드(`get()`)만 호출할 수 있게 됩니다.

이와 같이 제네릭 타입의 공변성이나 반공변성을 실제로 타입이 사용되는 코드 위치에서 지정하는 것을 **use-site variance**라고 합니다.

또한 `in`을 사용한 Type projection도 가능합니다.

```kotlin
fun fill(dest: Array<in String>, value: String) {
    ...
}
```

`Array<in String>`은 Java의 `Array<? super String>`에 해당됩니다.  
이는 `fill()`에 `CharSequence`의 배열이나 `Object`의 배열을 전달할 수 있음을 의미합니다.

---

### Star-projections

때때로 타입 인자에 대해 아무런 정보가 없지만, 안전한 방법으로 사용하고 싶을 수 있습니다.  
여기서 안전한 방법이란, 제네릭 타입에 '**Star-projections**'을 정의하는 것입니다.

'Star-projections'은 해당 제네릭 타입의 구체적인 인스턴스가 'Star-projections'의 하위 타입이 됩니다.

Kotlin은 'Star-projections'을 다음과 같은 목적으로 문법을 제공합니다.

`Foo<out T : TUpper>`와 같이 `T`가 상한 `TUpper`를 가진 공변 타입 파라미터일 때,  
`Foo<out T : Upper>`가 `Foo<*>`로 바뀌면, 이는 `Foo<out TUpper>`으로 해석됩니다.  
이는 `Foo<*>`를 통해 `TUpper` 타입의 값을 안전하게 얻을 수(`get()`) 있습니다.

```kotlin
class Foo<out T: Number> constructor(
    private val data: T
) {
    fun getData(): T = data
}

val fooInt: Foo<Int> = Foo(42)
val fooAny: Foo<*> = fooInt // OK
val data: Number = fooAny.getData() // OK
```

`Foo<in T>`와 같이 `T`가 반공변 타입 파라미터일 때,  
`Foo<in T>`가 `Foo<*>`로 바뀌면 이는 `Foo<in Nothing>`으로 해석됩니다.  
`Nothing`은 값이 존재할 수 없는 타입으로 의미되어 쓰기(`set()`)가 불가능해집니다.

```kotlin
class Foo<in T> {
    fun setData(value: T) { 
        println("value receiver : $value")
    }
}

val fooAny: Foo<*> = Foo<Any>()
// fooAny.setData(42) // Type mismatch. 'Required: Nothing' 'Found: Int' Compile Error
```

`Foo<T : TUpper>`와 같이 `T`가 상한 `TUpper`를 가진 불변 타입 파라미터라면,  
값 읽기 : `Foo<*>`와 `Foo<out TUpper>`와 동일하고,  
값 쓰기 : `Foo<*>`와 `Foo<in Nothing>`와 동일합니다.

```kotlin
class Foo<T> constructor(var data: T)

val fooInt: Foo<*> = Foo(42)
// fooInt.data = 22 // Compile Error
val readData: Any? = fooInt.data // OK
```

제네릭 타입이 여러 타입 파라미터를 가지고 있다면, 각각은 독립적으로 프로젝션될 수 있습니다.  
예를 들어 타입이 `interface Function<in T, out U>`로 선언되어 있다면,
다음과 같이 'Star-projections'을 사용할 수 있습니다.

- `Function<*, String>` means `Function<in Nothing, String>`.
- `Function<Int, *>` means `Function<Int, out Any?>`.
- `Function<*, *>` means `Function<in Nothing, out Any?>`.

----

### 상한

타입 파라미터 `T`가 가질 수 있는 타입의 범위를 제한하는 역할을 합니다.  
예를 들어 `<T : Any>`와 같이 타입 파라미터를 정의할 때, `T`는 `Any` 또는 그 하위 타입만 될 수 있습니다.
여기서 `Any`가 `T`의 '상한'이 됩니다.