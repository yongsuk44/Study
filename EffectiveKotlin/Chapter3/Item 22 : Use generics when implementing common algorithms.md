함수에 인자로 값을 전달하는 것처럼, 인자로 타입을 전달할 수 있다.  
타입 파라미터를 가지는 함수들을 제네릭 함수라고 부르며, 표준 라이브러리에서 잘 알려진 예시로는 `filter` 함수가 있다.

```kotlin
inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    val destination = ArrayList<T>()
    for (element in this) {
        if (predicate(element)) {
            destination.add(element)
        }
    }

    return destination
}
```

타입 파라미터는 컴파일러가 타입을 더 정확하게 확인하고 추론할 수 있도록 도와주기에, 프로그램을 안전하게 만들고 개발자들에게 더 나은 프로그래밍 경험을 제공한다.
예를 들어, 'filter' 함수 사용 시, 람다 표현식에서 컴파일러는 전달되는 인자가 컬렉션의 요소와 동일한 타입임을 알고 있으므로, 잘못된 사용을 방지해주고, IDE는 개발자에게 더 유용한 제안을 해줄 수 있다.

제네릭은 클래스와 인터페이스에 적용되어, `List<String>`, `Set<User>` 등과 같이 구체적인 타입만을 가지는 컬렉션을 생성할 수 있도록 한다.  
이 타입 정보는 컴파일 과정에서는 사라지지만, 개발하는 동안 컴파일러는 올바른 타입의 요소만 전달하도록 강제한다.
예를 들어, `MutableList<Int>`에는 `Int` 타입의 요소만 추가할 수 있고, `Set<User>`에서 반환되는 요소 타입이 `User`임을 컴파일러가 알려준다.
이처럼 타입 파라미터는 정적 타입 언어에서 큰 역할을 한다.

## Generic constraints

타입 파라미터는 특정 타입의 하위 타입으로만 제한될 수 있다는 중요한 특징을 가지고 있다.  
이러한 제약 조건은 콜론(:)과 함께 상위 타입을 지정하여 설정할 수 있으며, 지정된 타입은 이전에 정의된 타입 파라미터를 포함할 수도 있다.

```kotlin
// `T` 타입이 `Comparable<T>` 인터페이스를 구현해야 함. 즉, `T` 타입의 객체들은 서로 비교 가능해야 함을 의미함.
fun <T : Comparable<T>> Iterable<T>.sorted(): List<T> { /*...*/ }

// `C`는 `MutableCollection<in T>`의 하위 타입이어야 함.
// `T` 타입의 요소로 이루어진 `Iterable`의 모든 요소를 `C` 타입의 컬렉션에 추가한 후, 그 컬렉션을 반환함. 
fun <T, C : MutableCollection<in T>> Iterable<T>.toCollection(destination: C): C { /* ... */ }

// `T` 타입이 `ItemAdapter`의 하위 타입이어야 함.
class ListAdapter<T : ItemAdapter>(/* ... */) { /* ... */ }
```

타입에 제약 조건을 추가하는 것은 해당 타입의 인스턴스가 그 타입이 제공하는 모든 메서드를 활용할 수 있게 만드는 중요한 이점을 가진다.  
예를 들어, 다음과 같을 수 있다.

- `T`를 `Iterable<Int>`의 하위 타입으로 제약을 두면, `T` 타입의 인스턴스를 순회할 수 있고, 순회 과정에서 반복되는 요소들이 `Int` 타입임을 알 수 있다.
- `T`를 `Comparable<T>`로 제약을 두면, 해당 타입이 자신과 비교 가능함을 의미한다.
- `T`를 `Any`로 제약을 두면, 해당 타입이 'Nullable' 하지 않은 어떤 타입이라도 될 수 있음을 의미한다.

```kotlin
inline fun <T, R : Any> Iterable<T>.mapNotNull(transform: (T) -> R?): List<R> { 
    return mapNotNullTo(ArrayList<R>(), transform)
}
```

드문 경우지만, 한 타입에 대해 여러 상위 제약 조건을 설정해야 할 때, `where` 키워드를 사용하여 추가적인 제약을 설정할 수 있다.  
예를 들어, 어떤 동물이 `Animal`임과 동시에 `GoodTempered` 특성을 만족해야 할 때, 아래와 같이 작성할 수 있다.

```kotlin
fun <T : Animal> pet(animal: T) where T : GoodTempered { /* ... */ }
// or
fun <T> pet(animal: T) where T : Animal, T : GoodTempered { /* ... */ }
```