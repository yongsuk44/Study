# Item 22 : 공통 알고리즘 구현 시 제네릭을 사용

함수에 인자(Argument)를 전달하는 것처럼, 타입(Type)을 타입 인자(Type Arguemnt)로 전달할 수 있습니다.  
타입 인자를 받는 함수 (='타입 매개 변수'를 가지는 함수)를 제네릭(generic) 함수라고 합니다.

아래의 `filter`는 타입 매개변수 `T`를 가지는 예시 입니다.

```kotlin
inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    val destination = ArrayList<T>()
    for (element in this) {
        if (predicate(element)) destination.add(element)
    }
    
    return destination
}
```

컴파일러는 타입 매개변수를 통해 타입을 좀 더 정확하게 확인하고 추론할 수 있어 프로그램의 안전성을 향상시킬 수 있습니다.

> reified type을 제외한 제네릭은 일반적으로 JVM 타입 소거 특성으로 인해 타입 정보는 대부분 컴파일 시간에만 유용하고 런타임에는 제거 됩니다.

제네릭은 주로 클래스와 인터페이스에 도입되어, 구체적인 타입만을 가지는 컬렉션의 생성을 가능하게 합니다. 
예를 들어, `MutableList<Int>`는 `Int` 타입의 요소만을 가질 수 있도록 강제합니다. 
이런 방식으로, 컴파일러는 우리가 적절한 타입의 요소만을 컬렉션에 추가하도록 강제하고, 컬렉션에서 요소를 가져올 때는 올바른 타입을 반환하게 합니다.

---

## 제네릭 제약조건

타입 매개변수의 주요 특징 중 하나는 특정 타입의 서브타입으로 제약할 수 있을 수 있습니다.
슈퍼타입을 콜론 뒤에 위치하여 제약조건을 설정하고 이전 타입 매개변수를 포함 시킬 수 있습니다.

```kotlin
fun <T : Comparable<T>> sort(list: List<T>) { /*...*/ }

fun <T, C : MutableCollection<in T>> Iterable<T>.ToCollection(destination: C): C { /* ... */ }

class ListAdapter<T: ItemAdapter>(/* ... */) { /* ... */ }
```

제약조건을 가지는 경우, 해당 타입의 인스턴스가 해당 타입이 제공하는 모든 메소드를 사용할 수 있습니다.

아래는 그 예시들 입니다.

- `T`가 `Iterable<Int>`의 서브타입으로 제약된 경우, `T` 타입의 인스턴스를 순회할 수 있고, `Iterable`이 반환하는 요소들이 `Int` 타입이라는 것을 알 수 있습니다.
- `Comparable<T>`에 제약할 경우, 이 타입이 자기 자신과 '비교'될 수 있다는 것을 알 수 있습니다.
- 제약조건으로 많이 선택되는 다른 예는 `Any`이며, 이는 타입이 `Nullable`하지 않는 모든 타입이 될 수 있음을 의미 합니다.

```kotlin
inline fun <T, R : Any> Iterable<T>.mapNotNull(transform: (T) -> R?): List<R> { /* ... */ }
```

한 타입에 여러 개의 상위 제한을 설정해야 할 드문 경우에는, `where`를 사용하여 더 많은 제약을 설정할 수 있습니다.

```kotlin
fun <T : Animal> pet(animal: T) where T: GoodTempered { /* ... */ }

fun <T> pet(animal: T) where T: Animal, T: GoodTempered { /* ... */ }
```