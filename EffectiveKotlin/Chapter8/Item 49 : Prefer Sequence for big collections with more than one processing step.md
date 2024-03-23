# Item 49: 사이즈가 큰 컬렉션에서 여러 단계의 로직 처리 시 Sequence를 이용하여 처리

사람들은 종종 'Iterable'과 'Sequence'의 정의가 매우 유사하기 때문에 둘의 차이점을 놓치는 경우가 많다.

```kotlin
interface Iterable<out T> {
    operator fun iterator(): Iterator<T>
}

interface Sequence<out T> {
    operator fun iterator(): Iterator<T>
}
```

이들 사이의 유일한 형식적인 차이는 이름뿐이라고 할 수 있다.  
비록 'Iterable'과 'Sequence'는 완전히 다른 사용법(서로 다른 contract를 가짐)과 관련되어 있음에도 불구하고, 거의 모든 처리 함수가 다르게 동작한다.

'Sequence'는 지연 계산을 활용하므로, 'Sequence' 처리에 사용되는 'intermediate' 함수들은 실제 계산을 수행하지 않는다.  
대신, 새로운 연산을 추가하여 이전 'Sequence'를 'decorate' 하는 새로운 'Sequence'를 반환한다.  
이러한 모든 계산은 'toList', 'count'와 같은 'terminal' 연산이 실행될 때 평가된다.

반면, 'Iterable' 처리는 각 단계에서 'List'와 같은 컬렉션을 반환한다.

```kotlin
public inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    return filterTo(ArrayList<T>(), predicate)
}

public fun <T> Sequence<T>.filter(
    predicate: (T) -> Boolean
): Sequence<T> {
    return FilteringSequence(this, true, predicate)
}
```

결과적으로, 컬렉션 'processing' 연산은 사용 시 바로 실행된다.  
반면, 'Sequence'의 'processing' 함수는 'terminal' 연산(Sequence가 아닌 다른 결과를 반환하는 연산)이 수행될 때까지 실행되지 않는다.

예를 들어, 'Sequence'의 경우 'filter'는 'intermediate' 연산에 해당하므로,
실제로 계산을 수행하지 않고 새로운 'processing' 단계를 추가하여 'Sequence'를 'decorate' 한다.
실제 계산은 'toList'와 같은 'terminal' 연산에서 진행된다.

```kotlin
val seq = sequenceOf(1, 2, 3)
val filtered = seq.filter {
    print("f$it")
    (it % 2) == 1
}

print(filtered)                         // kotlin.sequences.FilteringSequence@7b23ec81
val asList = filtered.toList()          // f1 f2 f3
print(asList)                           // [1, 3]

val list = listOf(1, 2, 3)
val listFiltered = list.filter {        // f1 f2 f3
    print("f$it")
    (it % 2) == 1
}

print(listFiltered)                     // [1, 3]
```

Kotlin에서 'Sequence'가 지연 처리되는 사실은 몇 가지 중요한 장점을 가진다.

- 연산의 자연스러운 순서를 유지한다. - 개발자가 예상한대로 코드가 실행됨
- 최소한의 연산만 수행한다. - 필요한 만큼의 연산만 수행되어 효율적임
- 무한한 데이터 스트림을 표현할 수 있다. - 끊임없는 데이터를 생성하는 시나리오에서 유용함
- 매 단계마다 컬렉션을 생성할 필요가 없다. - 메모리 사용량이 줄고 성능이 개선됨

---

## Order is important

'Iterable'과 'Sequence' 처리의 구현 방식에 따라, 그들의 연산 순서가 다르다.  
'Sequence' 처리 과정에서는 첫 번째 요소에 모든 연산을 적용한 후, 그 다음 요소로 넘어가 이 과정을 반복한다.  
이러한 접근 방식을 'element by element' 또는 'lazy order'라고 한다.

반면, 'Iterable' 처리에서는 첫 번째 연산을 전체 컬렉션에 적용하고, 그 다음 연산으로 넘어가는 식으로 진행된다.  
이를 'step by step' 처리 또는 'eager order'라고 한다.

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
```

주목할 점은, 만약 우리가 컬렉션 처리 함수 없이 전통적인 반복문과 조건문을 사용하여 위 연산들을 구현한다면,
'Sequence processing'과 같은 'element by element'를 따르게 될 것이라는 점이다.

```kotlin
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

따라서, 'Sequence processing'에 사용되는 'element by element'가 더 자연스러운 방식으로 여겨진다.  
이 접근법은 저수준의 컴파일러 최적화의 가능성을 열어준다. 나중에 'Sequence processing'은 기본적인 루프와 조건문으로 최적화될 수 있을 것이다.

---

## Sequences do the minimal number of operations

컬렉션 전체를 매번 처리할 필요는 없다.   
예를 들어, 수백만 개의 원소를 가진 컬렉션을 가지고 있고, 이를 처리한 뒤 단지 처음 10개의 원소만 필요하다고 가정해보자. 나머지 모든 원소를 처리하는 것은 비효율적이다.
'Iterable' 처리에는 'intermediate' 연산이 포함되지 않기에, 컬렉션의 모든 요소가 마치 최종 결과로 반환되어야 할 것처럼 처리된다.
반면, 'Sequence'는 그럴 필요가 없다. 그래서 결과를 얻기 위해 필요한 최소한의 연산만을 수행하게 된다.

몇 가지 'processing' 단계가 있고, 'find'로 'processing'을 끝내는 예제를 살펴보자.

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

특정 'intermediate processing'을 거치며 'terminal' 연산에서 모든 요소를 반복할 필요가 없는 경우, 'Sequence'를 사용하는 것이 처리 성능을 크게 향상 시킬 수 있다.
이는 표준 컬렉션 처리와 유사하게 보이지만, 성능 측면에서는 훨씬 우수하다. 'first', 'find', 'take', 'any', 'all', 'none', 'indexOf'와 같은 연산에서 이러한 차이가 더욱
두드러진다.

---

## Sequences can be infinite

'Sequence'가 요청에 따라 'processing' 될 수 있다는 사실 덕분에, 'Infinite sequence'를 만들 수 있다.  
'Infinite sequence'를 만드는 일반적인 방법은 'generateSequence'나 'sequence'와 같은 'sequence generator'를 사용하는 것이다.  
'generateSequence'를 사용할 경우 첫 번째 요소와 그 다음 요소를 어떻게 계산할지를 정의하는 함수가 필요하다.

```kotlin
generateSequence(1) { it + 1 }          //  first element, calculate the next element
    .map { it * 2 }
    .take(10)
    .forEach { print("$it ") }
// Prints : 2 4 6 8 10 12 14 16 18 20, 
```

두 번째로 언급된 'sequence generator'인 'sequence'는 요구사항에 따라 다음 숫자를 생성하는 'suspending function', 즉 'Coroutine'을 활용한다.  
다음 숫자를 요청할 때마다, 'sequence builder'는 'yield'를 통해 값을 반환할 때까지 실행을 계속한다.  
값이 반환된 후에는 다음 값을 요청받을 떄까지 실행이 일시 중단된다.

예를 들어, 피보나치 수열의 다음 숫자들을 나타내는 무한 리스트를 만드는데 이 방식을 사용할 수 있다.

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

'infinite sequence'는 특정 시점에서 요소의 수가 제한되어야 한다는 점을 유의해야 한다.  
이를 고려하지 않으면 무한 루프에 빠져 프로그램이 멈출 수 있다.

```kotlin
print(fibonacci.toList()) // Runs forever
```

따라서 'take'와 같은 연산을 사용하여 제한하거나, 모든 요소가 필요하지 않은 'first', 'find', 'any', 'all', 'none', 'indexOf'와 같은 'terminal' 연산을 사용해야
한다.
이러한 연산들은 전체 요소를 처리할 필요가 없어 'Sequence'와 잘 어울린다.
그러나, 'any', 'all', 'none' 같은 연산들은 'infinite sequence'에 사용할 경우 무한 루프에 빠질 위험이 있다.
따라서, 보통 'take'를 사용해 요소의 수를 제한하거나, 'first'를 사용해 첫 번재 요소만을 요청한다.

---

## Sequences do not create collections at every processing step

일반적인 컬렉션 처리 방식은 처리의 각 단계마다 새로운 컬렉션(주로 'List')을 반환한다.  
이 방식은 각 단계마다 결과물이 바로 사용 가능하거나 저장될 수 있다는 장점을 가진다.  
하지만, 이 과정에서 발생하는 비용은 무시할 수 없다. 처리의 각 단계마다 컬렉션이 새롭게 생성되고, 데이터로 채워져야 하기 떄문이다.

```kotlin
// 2 collections created under the hood
numbers
    .filter { it % 10 == 0 }
    .map { it * 2 }
    .sum()

// No collections created
numbers
    .asSequence()
    .filter { it % 10 == 0 }
    .map { it * 2 }
    .sum() 
```

이 문제는 크거나 복잡한 컬렉션을 다룰 때 더욱 문제가 될 수 있다. 예를 들어, 파일 읽기와 같은 흔하면서도 극단적인 사례를 살펴보자.  
파일은 기가바이트 단위의 크기를 가질 수 있다. 이런 파일 데이터를 처리하는 각 단계에서 모든 데이터를 컬렉션에 저장하려고 하면, 엄청난 양의 메모리 낭비가 발생한다.
이러한 이유로 파일 처리 시 기본적으로 'Sequence'를 사용하는 것이다.

아래 예시는 범죄 데이터베이스를 공유하여 특정 범죄가 몇 건인지 찾아보는 과정이라고 가정해보자.  
아래는 컬렉션 처리를 사용했을 떄의 기본적인 해결 방식이다.

```kotlin
// Bad solution, Do not use collections for possibly big files
File("crimes.csv")
    .readLines()                                    // Returns List<String>
    .drop(1)                                        // Drop descriptions of the columns
    .mapNotNull { it.split(",").getOrNull(6) }      // Find descriptions
    .filter { "CANNABIS" in it }                    // Description is in uppercase
    .count()
    .let(::println)
```

위 결과는 'OutOfMemoryError'를 발생시킬 수 있다.
이유는 하나의 컬렉션을 생성하고, 그 다음에 세 가지의 'intermediate' 연산을 거쳐 총 4개의 컬렉션을 만든다.  
이 중 3개는 이 데이터 파일의 대부분을 담고 있어, 파일 크기가 1GB라면 3GB 이상의 메모리를 소모한다.
이는 막대한 메모리 낭비를 의미하며, 올바른 구현 방법은 'Sequence'를 사용하는 것이다.

이를 위해 'useLines'를 사용하여 항상 단일 라인만을 대상으로 처리하는 것이 좋다.

```kotlin
File("crimes.csv").useLines { lines: Sequence<String> ->
    lines
        .drop(1)                                        // Drop descriptions of the columns
        .mapNotNull { it.split(",").getOrNull(6) }      // Find descriptions
        .filter { "CANNABIS" in it }                    // Description is in uppercase
        .count()
        .let(::println)                                 // Print the result
}
```

컬렉션이 반드시 크고 무거울 필요는 없다.  
그렇지만, 매 단계마다 새로운 컬렉션을 만드는 사실 자체가 이미 하나의 비용이며, 이 비용은 원소의 수가 많은 컬렉션을 다룰 때 더욱 두드러진다.  
많은 단계를 거친 후에 생성된 컬렉션은 예상된 크기로 초기화되므로 원소를 추가할 때 그저 다음 위치에 배치하기만 하면 되기에 비용의 차이가 매우 큰건 아니다.  
그럼에도 불구하고, 이 차이는 중요하며, 바로 이 점이 하나 이상의 'processing' 단계를 거치는 큰 컬렉션에 대해서는 'Sequence'를 사용하는 것이 좋은 이유이다.

하나 이상의 'processing' 단계는 컬렉션 처리를 위한 단 하나의 함수만 사용하는 것이 아닌, 여러 함수를 사용하는 경우를 말한다.  
이런 관점에서 아래 두 가지 함수를 비교해보자.

```kotlin
fun singlesStepListProcessing(): List<Product> {
    return productsList.filter { it.bought }
}

fun singleStepSequenceProcessing(): List<Product> {
    return productsList.asSequence().filter { it.bought }.toList()
}
```

위 두 함수를 비교해보면 성능상 거의 차이가 없음을 알 수 있다.
실제로는 리스트를 간단히 처리하는 경우가 더 빠른데, 그 이유는 'filter' 함수가 'inline'으로 처리되기 때문이다.

하지만, 'filter' 다음에 'map'을 사용하는 것과 같이, 두 단계 이상의 'processing'을 포함하는 함수를 비교할 떄는 큰 컬렉션에서 차이가 확연히 드러난다.
이 차이를 확인하기 위해, 5000개의 제품으로 두 단계와 세 단계의 'processing'를 포함하는 전형적인 처리 방식을 비교해보자.

```kotlin
fun twoStepListProcessing(): List<Double> {
    return productsList
        .filter { it.bought }
        .map { it.price }
}

fun twoStepSequenceProcessing(): List<Double> {
    return productsList
        .asSequence()
        .filter { it.bought }
        .map { it.price }
        .toList()
}

fun threeStepListProcessing(): Double {
    return productsList
        .filter { it.bought }
        .map { it.price }
        .average()
}

fun threeStepSequenceProcessing(): Double {
    return productsList
        .asSequence()
        .filter { it.bought }
        .map { it.price }
        .average()
}
```

성능 향상을 얼마나 기대할 수 있을지 예측하는 건 쉽지 않다.  
하지만, 한 단계 이상으로 데이터를 처리하는 일반적인 컬렉션 작업에서는 최소한 몇 천개의 원소가 있는 경우 대략 20% ~ 40% 사이의 성능 향상을 기대할 수 있다.

---

## When aren't sequences faster?

일부 연산들은 전체 컬렉션을 처리해야만 하므로, 'Sequence'의 사용으로부터 특별한 이점을 얻지 못한다.  
Kotlin 표준 라이브러리에서 'sorted' 함수가 이에 해당된다.

'sorted' 함수는 최적화된 구현 방식을 채택하고 있다.'Sequence'를 리스트로 변환한 뒤, Java 표준 라이브러리의 'sort'를 사용한다.  
이 변환 과정이 컬렉션에서의 동일 작업에 비해 조금 더 시간이 소요된다는 점이 단점이지만,
'Iterable'이 컬렉션이나 배열이 아닐 경우에는 어차피 변환 과정이 필요하기 때문에, 그 차이는 크지 않다.

'Sequence'에 'sorted'와 같은 메서드를 포함시켜야 하는지에 대한 논의는 활발하다.  
다음 요소를 계산하기 위해 모든 요소가 필요한 메서드를 갖는 'Sequence'가 부분적으로만 지연 처리되는 특성 때문에 'Infinite Sequence'에서는 작동하지 않는다.
이런 메서드가 추가된 이유는 'sorted'는 매우 인기 있는 기능이며, 이 방식을 통해 사용하기가 더 쉽기 때문이다.
하지만, Kotlin 개발자들은 이 메서드의 단점을 잘알고 있어야 하며, 특히 'Infinite Sequence'에서 사용할 수 없다는 것을 명심해야 한다.

```kotlin
generateSequence(0) { it + 1 }.take(10).sorted().toList() // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
generateSequence(0) { it + 1 }.sorted().take(10).toList() // Infinite time. Does not return 
```

'sorted'는 컬렉션에서 'Sequence' 보다 처리가 더 빨리 이루어지는 드문 경우 중 하나이다.  
그렇지만, 몇 단계의 'processing' 과정과 하나의 'sorted' 함수(혹은 전체 컬렉션을 대상으로 해야 하는 다른 함수)를 사용한다면,
'Sequence processing' 방식을 통해 성능 개선을 기대할 수 있다.

```kotlin
// Benchmarking measurement result: 150 482ns
fun productsSortAndProcessingList(): Double =
    productsList
        .sortedBy { it.price }
        .filter { it.bought }
        .map { it.price }
        .average()

// Benchmarking measurement result: 96 811ns
fun productsSortAndProcessingSequence(): Double =
    productsList
        .asSequence()
        .sortedBy { it.price }
        .filter { it.bought }
        .map { it.price }
        .average()
```

---

## What about Java stream

Java8은 컬렉션을 처리하기 위해 'Stream'을 도입했다. 이 'Stream'은 Kotlin의 'Sequence'와 매우 유사하게 동작한다.

```kotlin
producesList.asSequence()
    .filter { it.bought }
    .map { it.price }
    .average()

produecsList.stream()
    .filter { it.bought }
    .mapToDouble { it.price }
    .average()
    .orElse(0.0)
```

Java8의 'Stream'은 'processing'의 마지막(terminal) 단계에서 결과가 수집되는 지연 처리 방식으로 동작한다.  
이와 Kotlin Sequence 사이에는 몇 가지 큰 차이점이 있다.

1. 'Sequence'는 확장 함수를 통해 정의될 수 있어 더 다양한 'processing' 기능을 제공하며 사용하기도 더 쉽다.  
   예를 들어, 'collect(Collectors.toList())' 대신 'toList()'를 사용하여 간편하게 수집할 수 있다.
2. 'parallel' 함수를 사용하여 'Stream'을 병렬로 처리할 수 있다. 이는 여러 CPU 코어를 활용해 상당한 성능 향상을 얻을 수 있다.
3. 'Sequence'는 Kotlin/JVM, Kotlin/JS, Kotlin/Native 모듈에서 사용할 수 있지만, 'Stream'은 Kotlin/JVM에서만, 그것도 JVM 버전이 9 이상일 경우에만 사용할
   수 있다.

병렬 모드를 활용하지 않는 경우, 'Stream'과 'Sequence' 중 어느 것이 더 효율적이라고 단정지을 수 없다.  
하지만, 병렬 모드를 통해 상당한 성능 향상을 기대할 수 있는 계산 집약적인 작업에 한해서는 'Stream'을 선택적으로 사용하는것이 좋다.
그렇지 않은 경우에는 다양한 플랫폼이나 공통 모듈에서 사용할 수 있으며, 일관성 있고 깔끔한 코드를 유지할 수 있는 'Sequence'를 사용하는 것이 좋다.