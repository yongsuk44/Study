# Item 24 : 제네릭 타입의 공변성과 반공변성 고려

```kotlin
class Cup<T>
```

타입 파라미터 `T`는 아무런 변위 수식어(`out` or `in`)가 없으로므로 기본적으로 불변입니다.  
이는 해당 제네릭 클래스로 생성된 타입 사이에는 어떤 관계도 없음을 의미 합니다.  
예를 들어, `Cup<Int>`, `Cup<Number>`, `Cup<Any>`, `Cup<Nothing>` 사이에는 어떠한 관계도 없습니다.

```kotlin
fun main() {
    val anys: Cup<Any> = Cup<Int>() // Error Type mismatch
    val nothings: Cup<Nothing> = Cup<Int>() // Error Type mismatch
}
```

## 변위 수식어(variance modifier) `out` `in`
위와 같은 관계가 필요한 경우, 변위 수식어 `out` 또는 `in`을 사용해야 합니다.

### out
`out` 수식어는 타입 파라미터를 공변(covariant)으로 만들어 줍니다.  
`Cup`이 `out` 수식어를 사용하여 공변이 되고, A가 B의 하위 타입일 경우 타입 `Cup<A>`는 `Cup<B>`의 하위 타입이 됩니다.

```kotlin
class Cup<out T>
open class Dog
class Puppy: Dog()

fun main(args: Array<String>) {
    val dogs: Cup<Dog> = Cup<Puppy>() // OK
    val puppies: Cup<Puppy> = Cup<Dog>() // Error Type mismatch
    
    val anys: Cup<Any> = Cup<Int>() // OK
    val nothings: Cup<Nothing> = Cup<Int>() // Error Type mismatch
}
```

### in

`out`의 반대 효과는 `in` 수식어를 사용하여 얻을 수 있으며, 이는 타입 파라미터를 반공변(contravariant)으로 만듭니다.  
즉, A가 B의 하위 타입이고, `in`을 사용하여 `Cup`이 반공변이면 `Cup<A>`는 `Cup<B>`의 상위 타입이 됩니다.


```kotlin
class Cup<in T>
open class Dog
class Puppy: Dog()

fun main(args: Array<String>) {
    val dogs: Cup<Dog> = Cup<Puppy>() // Error Type mismatch
    val puppies: Cup<Puppy> = Cup<Dog>() // OK
    
    val anys: Cup<Any> = Cup<Int>() // Error Type mismatch
    val nothings: Cup<Nothing> = Cup<Int>() // OK
}
```