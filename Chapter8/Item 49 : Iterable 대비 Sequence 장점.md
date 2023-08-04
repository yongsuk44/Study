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

---

## Sequences는 무한할 수 있음

`Sequences`는 요청에 따라 처리하기에 무한할 수 있습니다.  
무한 `Sequence`를 만드는 일반적인 방법은 `generateSequence`나 Coroutine의 `sequence` 같은 생성자를 사용할 수 있습니다.

아래 예제는 시작 요소와 다음 요소를 계산하는 함수를 지정하여 `Sequence`를 생성한 예제 입니다.

```kotlin
generateSequence(1) { it + 1 }
    .map { it * 2 }
    .take(10)
    .forEach { print("$it ") }
// Prints : 2 4 6 8 10 12 14 16 18 20, 
```

`sequence`는 요청에 따라 다음 번호를 생성하는 `suspend` 함수(coroutine)을 사용합니다.

다음 번호를 요청할 때, `sequence` 생성자는 `yield`를 사용하여 값을 출력할 때까지 실행됩니다.
그 후 실행이 중단되고, 다른 번호를 요청할 때까지 상태를 유지합니다.

아래는 무한한 피보나치 수열 리스트 예제입니다.

```kotlin
val fibonacci = sequence {
    yield(1)
    var current = 1
    var prev = 1
    while (true) {
        yield(current)
        val temp = prev
        prev = current
        current += temp
    }
}

print(fibonacci.take(10).toList())
// Prints : [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```

무한한 `Sequence`는 어느 시점에서 요소의 수를 제한해야 합니다. 이를 주의하지 않으면 무한 루프에 빠져 프로그램이 멈추게 됩니다.

```kotlin
print(fibonacci.toList()) // Runs forever
```

그러므로 `take`와 같은 연산을 사용하거나, `first`, `find`, `any`, `all`, `none`, `indexOf`와 같이 모든 요소를 필요로 하지 않는 터미널 연산을 사용해야 합니다.

기본적으로 위 연산들은 모든 요소를 처리할 필요가 없기에 `Sequence`에 사용하기에 효율적인 연산들입니다.   
그러나 위 연산들은 대부분 무한 루프에 빠지기 쉽다는 점을 인지하고 사옹하여야 합니다.

무한 `Sequence`에서 `any`는 `true`를 반환하거나 무한히 실행될 수 있습니다.
또한 `all`과 `none`도 마찬가지로 무한한 컬렉션에서는 `false`만을 반환할 수 있습니다.

이에따라 일반적으로 `take`를 이용하여 요소의 수를 제한하거나, `first` 등을 이용하여 첫 번째 요소만을 요청하여 사용하는 것이 좋습니다.

---

## 시퀀스는 모든 처리 단계에서 컬렉션을 생성하지 않음

표준 컬렉션 처리 함수들은 각 단계에서 새로운 컬렉션을 반환하며 이들은 대부분 리스트 형태입니다.  
이는 장점이 될 수 있습니다. 각 연산 후 리스트를 사용하거나 저장할 수 있기 때문입니다.

그렇지만 이러한 컬렉션들은 모든 단계에서 생성되고 데이터로 채워져야 하기에 비용이 발생합니다.

```kotlin
numbers.filter { it % 10 == 0 }.map { it * 2 }.sum()
// 2개의 컬렉션이 내부에서 생성

numbers.asSequence().filter { it % 10 == 0 }.map { it * 2 }.sum()
// 컬렉션을 생성하지 않음 
```

이러한 비용은 사이즈 큰 컬렉션을 다룰 때 문제가 될 수 있습니다.

예를 들어 파일을 읽는 경우, 파일의 크기가 GB 단위로 나갈 수 있습니다. 
이러한 경우 모든 처리 단계에서 모든 데이터를 컬렉션에 할당하는 것은 엄청난 메모리 낭비로 볼 수 있습니다.

그렇기에 기본적으로 파일을 처리하기 위해서는 `Sequence`를 사용합니다.

아래 예시는 1GB의 범죄 파일을 컬렉션을 통해 처리하는 잘못된 솔루션 예제 입니다. 

```kotlin
File("crimes.csv")
    .readLines() 
    .drop(1) 
    .mapNotNull { it.split(",").getOrNull(6) } 
    .filter { "CANNABIS" in it }
    .count()
    .let(::println)
```

위 코드는 컬렉션을 생성하고 3개의 중간 연산을 거쳐 총 4개의 컬렉션을 생성하게 됩니다. 
이 중 3개는 1GB를 차지하는 데이터 파일의 대부분을 포함하고 있으므로 총 3GB 이상의 메모리를 소비해야 합니다.
이러한 처리는 엄청난 메모리 낭비입니다. 위에서 설명한것 처럼 올바른 구현은 `Sequence`를 사용해야 합니다.

아래 예제는 `useLines`를 사용하여 처리하는 예제 입니다. 

```kotlin
File(crimes.csv).useLines { lines: Sequence<String> ->
    lines
        .drop(1) 
        .mapNotNull { it.split(",").getOrNull(6) } 
        .filter { "CANNABIS" in it } 
        .count()
        .let(::println)
}
```

첫번째 구현에서 컬렉션 처리를 하면 약 8s가 소요되었고, 두번째 구현에서는 `Sequence`를 사용하여 약 3s가 소요되었습니다.
위 결과와 같이 더 큰파일에 대한 `Sequence`를 사용하는 것은 메모리 뿐만 아니라 성능에도 큰 영향을 미칩니다.

또한 둘 이상의 중간 연산이 있는 경우, 큰 컬렉션에 대한 `Sequence` 처리는 `List` 처리보다 성능이 뛰어나며, 약 20-40%의 성능 향상을 기대할 수 있습니다.

---

## sequences 처리가 더 느린 경우

몇몇 연산에서는 전체 컬렉션에 대한 작업을 해야하기에 `Sequence`를 사용하더라도 이익을 볼 수 없는 경우가 있습니다.

대표적으로 `sorted`가 있으며, `sorted`는 `Sequence`를 리스트로 누적 한 다음 Java 표준 라이브러리인 `sort`를 사용하여 구현합니다. 
이 과정에서 추가적인 시간이 걸리지만, `Iterable`이 컬렉션이나 배열이 아니라면 그 차이는 크지 않습니다.

`Sequence`가 `sorted`와 같은 연산을 해야하는 것에 대해 의견이 분분합니다. 
왜냐하면 다음 요소를 계산하기 위해 모든 요소가 필요한 연산을 가진 `Sequence`는 부분적으로만 지연 연산을 하고 `Infinite Sequence`에서는 동작하지 않기 때문입니다.

Kotlin 개발자들은 이 단점을 알고 있어야 하며, 특히 `Infinite Sequence`에서 사용할 수 없다는 것을 명심해야 합니다.

위와 같은 의견이 갈리는 상황에도 불구하고 `sorted`는 인기 있는 연산이기에 추가되었고, 아래와 같은 방식으로 처리하는 것을 권장합니다.

```kotlin
generateSequence(0) { it + 1 }.take(10).sorted().toList() // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
generateSequence(0) { it + 1 }.sorted().take(10).toList() // Infinite time. Does not return 
```

`sorted` 연산은 `Sequence` 처리가 더 느린 예시 입니다. 그럼에도 몇몇 처리 단계를 거치는 경우에는 `Sequence`를 사용하는 것이 성능에 더 좋습니다.

```kotlin
// Benchmarking measurement result: 150482ns
fun productsSortAndProcessingList(): Double =
    productsList
        .sortedBy { it.price }
        .filter { it.bought }
        .map { it.price }
        .average()

// Benchmarking measurement result: 96811ns
fun productsSortAndProcessingSequence(): Double =
    productsList
        .asSequence()
        .sortedBy { it.price }
        .filter { it.bought }
        .map { it.price }
        .average()
```