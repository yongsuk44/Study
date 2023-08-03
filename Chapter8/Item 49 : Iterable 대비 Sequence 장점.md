# Item 49: 사이즈가 큰 컬렉션에서 여러 단계의 로직 처리 시 Sequence를 이용하여 처리

`Iterable`과 `Sequence`은 정의가 거의 동일하기에 둘의 차이점을 쉽게 놓칠 수 있습니다.

```kotlin
interface Iterable<out T> {
    operator fun iterator(): Iterator<T>
}

interface Sequence<out T> {
    operator fun iterator(): Iterator<T>
}
```

위 인터페이스 처럼 유일한 차이점을 이름뿐이지만, 실제 사용법은 다릅니다.
`Sequence`는 `lazy` 반복자로 연산이 즉시 실행되지 않고, 필요할 때까지 중간 연산이 지연되는 것을 의미합니다.

대신, 중간 연산은 이전 작업에 새 작업을 추가하는 새로운 `Sequence`를 반환합니다.
실제 연산은 터미널 연산(`toList()` 혹은 `count()`)이 호출될 때만 발생됩니다.

반면에, `Iterable`은 모든 중간 연산에서 컬렉션을 반환합니다.

```kotlin
public inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    return filterTo(ArrayList<T>(), predicate)
}

public fun <T> Sequence<T>.filter(predicate: (T) -> Boolean): Sequence<T> {
    return FilteringSequence(this, true, predicate)
}
```

이러한 차이로 인해, `Sequence`의 경우 중간 연산이 많아도 최종 연산이 호출될 때까지 연산이 수행되지 않아 마지막 터미널 연산에서만 호출되고,
`Iterable`의 경우 각 단꼐에서 호출됩니다.

```kotlin
val seq = sequenceOf(1,2,3)
val filtered = seq.filter { 
    print("f$it")
    (it % 2) == 1 
}

print(filtered)
val asList = filtered.toList()
print(asList)

val list = listOf(1,2,3)
val listFiltered = list.filter { 
    print("f$it")
    (it % 2) == 1 
}

print(listFiltered)
```

Kotlin에서 `Sequence`는 `Iterable`과 비교하여 다음과 같은 장점을 가집니다.

- 작업 순서 유지 : 연산이 정의된 순서대로 실행되는 것을 보장합니다.
- 최소 작업 수행 : 필요한 만큼의 작업만 수행되어 효율적입니다.
- 무한 가능 : `Sequence`는 무한할 수 있으며, 터미널 연산이 호출될 때까지 실행되지 않습니다.
- 중간 컬렉션 불필요 : 각 단계에서 새로운 컬렉션을 만들 필요가 없으므로 메모리 효율이 높아집니다.

이러한 이점으로 컬렉션의 크기와 복잡성이 커질수록 더욱 중요해집니다.
따라서 2개 이상의 중간 연산자가 있는 큰 컬렉션을 다룰 때 `Sequence`를 사용하는 것이 효율적입니다.
