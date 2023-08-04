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
val seq = sequenceOf(1, 2, 3)
val filtered = seq.filter {
    print("f$it")
    (it % 2) == 1
}

print(filtered)
val asList = filtered.toList()
print(asList)

val list = listOf(1, 2, 3)
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

---

## 순서의 중요성

### Sequence 처리 (`element-by-element` 또는 `lazy order`)

- lazy order : 시퀀스 처리는 연산을 "lazy"하게 처리합니다. 이 말은, 각 요소에 대해 모든 연산이 한 번에 수행되고, 그 다음 요소로 넘어간다는 것을 의미합니다.
- 순서 : 첫 번째 요소를 가져와 연산을 수행하고, 다음 요소로 넘어갑니다. 예를 들어, `filter` -> `map` -> `forEach` 순서로 첫 번째 요소에 대해 수행한 후, 다음 요소로 진행합니다.
- 자연스러움: 기본 `loop`를 통한 방식과 유사하며, 더 자연스럽게 코드를 이해할 수 있습니다.
- 컴파일러 최적화 가능성: 요소별 순서는 기본 `loop`의 조건으로 최적화될 수 있어 효율성이 향상될 가능성이 있습니다. 

### Iterable 처리 (`step-by-step` 또는 `eager order`)

- eager order : 이터러블 처리는 "즉시" 처리됩니다. 첫 번째 연산이 전체 컬렉션에 적용되고, 그 다음 연산으로 넘어갑니다.
- 순서 : 첫 번째 연산을 전체 컬렉션에 적용한 뒤, 그 다음 연산으로 넘어갑니다. 예를 들어, `filter`가 모든 요소에 대해 수행된 후, `map` -> `forEach` 순으로 진행됩니다.
- 연산의 집중 : 특정 연산에 집중해서 한 번에 처리하기 때문에, 각 단계별 컬렉션의 변화를 파악할 수 있습니다.

```kotlin
sequenceOf(1, 2, 3)
    .filter { print("f$it"); (it % 2) == 1 }
    .map { print("m$it"); it * 2 }
    .forEach { print("e$it") }
// Prints : f1 m1 e2 f2 f3 m3 e6

listOf(1, 2, 3)
    .filter { print("f$it"); (it % 2) == 1 }
    .map { print("m$it"); it * 2 }
    .forEach { print("e$it") }

// Prints : f1 f2 f3 m1 m3 e2 e6

for (e in listOf(1, 2, 3)) {
    print("f$e, ")

    if (e % 2 == 1) {
        print("m$e, ")
        val m = e * 2
        print("e$m, ")
    }
}

// Prints : f1, m1, e2, f2, f3, m3, e6,
```

---

## Sequence는 최소한의 연산만 수행

`Sequence`는 전체 컬렉션을 최소한의 연산을 수행하여 결과를 생성합니다. 

만약 1M개의 요소를 지닌 컬렉션을 연산하여 처음 10개의 요소만 갖고오도록 한다면, 10개를 제외한 다른 요소의 연산 비용은 불필요합니다. 
이때 `Iterable`의 경우 모든 연산이 전체 컬렉션을 반환하는 방식으로 처리되기에 효율적이지 않기에 `Sequence`를 사용하는 것이 좋습니다.

아래 예시를 보면, 몇 단계의 처리가 있고 `find`로 처리를 끝내는 것을 볼 수 있습니다.

```kotlin
(1..10).asSequence()
    .filter { print("f$it, "); it % 2 == 1 }
    .map { print("m$it, "); it * 2 }
    .find { it > 5 }
// Print : f1, m1, f2, f3, m3

(1..10)
    .filter { print("f$it, "); it % 2 == 1 }
    .map { print("m$it, "); it * 2 }
    .find { it > 5 }
// Print : f1, f2, f3, f4, f5, f6, f7, f8, f9, f10, m1, m3, m5, m7, m9
```

중간 처리(intermediate processing) 단계가 있고 터미널 연산(위 코드의 경우 `find`)이 모든 요소를 반복할 필요가 없을 때 `Sequence`를 사용하면 처리 성능이 더 좋습니다.
이러한 연산의 예로는 `first`, `find`, `take`, `any`, `all`, `none`, `indexOf` 등이 있습니다.