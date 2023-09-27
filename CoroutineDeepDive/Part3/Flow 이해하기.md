# Understanding Flow

Flow는 여러 연산들의 실행 정의를 나타내는 개념입니다. 이는 중단 람다 표현식(suspending lambda expression)이라는 것과 유사한데,
중단 람다 표현식은 특정 연산이나 작업을 잠시 중단하고 나중에 다시 시작할 수 있게 해주는 기능을 가진 코드 블록을 의미합니다.

람다 표현식은 간단히 말하면, 함수처럼 동작하는 코드 블록으로 `{ x, y -> x + y }`는 2개의 인수를 받아 그들의 합을 반환하는 람다입니다.  
이러한 람다 표현식은 한 번 정의하면 여러 번 재사용할 수 있습니다.

```kotlin
fun main() {
    val f: () -> Unit = {
        print("A")
        print("B")
        print("C")
    }
    f() // ABC
    f() // ABC
}
```

여기서 람다 표현식에 `suspend` 키워드를 사용하여 중단 가능한 람다를 만들 수 있습니다.  
이 `suspend` 키워드는 코루틴 내에서만 실행될 수 있으며, `delay`와 같은 함수를 사용해 람다의 실행을 일시적으로 중지할 수 있습니다.

그러나 이러한 중단 람다 표현식은 각 호출이 순차적으로 이루어져야 하기에, 한 번 호출되면 그것이 완료될 떄까지 다른 호출을 시작하면 안됩니다.

```kotlin
suspend fun main() {
    val f: suspend () -> Unit = {
        print("A")
        delay(1000)
        print("B")
        delay(1000)
        print("C")
    }
    f()
    f()
}
// A
// 1s delay
// B
// C
// A
// 1s delay
// B
// 1s delay
// C
```

람다식은 단순한 연산 뿐만 아니라 함수를 파라미터로 받을 수 있습니다.
예를 들어 람다식 내부에서 어떤 값을 '방출'하는 동작을 위해 함수를 파라미터로 받아 사용할 수 있습니다.

아래 예제처럼 `emit` 파라미터는 '방출'을 담당하는 함수를 나타냅니다.  
위에서 `f` 람다식을 호출할 때, `emit` 함수를 정의한 람다식을 파라미터로 전달하여 `f` 내부에서 원하는 값이나 결과를 '방출'하도록 할 수 있습니다.

```kotlin
suspend fun main() {
    val f: suspend ((String) -> Unit) -> Unit = { emit ->
        emit("A")
        emit("B")
        emit("C")
    }
    f { print(it) } // ABC
    f { print(it) } // ABC
}
```

위처럼 `emit`도 중단 함수여야 하기에, 파라미터로 받는 람다식의 타입이 상당히 복잡해질 수 있습니다.

이러한 복잡성을 줄이기 위해 `FlowCollector`라는 함수 인터페이스를 도입하며, 이 인터페이스 내에는 `emit`이라는 중단 함수 형태의 추상 메서드가 포함됩니다.  
그리고 이 `FlowCollector`를 사용하면 람다식의 타입을 명시적으로 나타낼 필요가 없어집니다.

함수형 인터페이스의 장점은 람다식으로 표현될 수 있다는 것입니다.  
따라서 `f`를 호출할 때, `FlowCollector`의 인스턴스를 전달할 필요 없이 직접 람다식을 사용하여 호출할 수 있습니다.

```kotlin
fun interface FlowCollector {
    suspend fun emit(value: String)
}

suspend fun main() {
    val f: suspend (FlowCollector) -> Unit = {
        it.emit("A")
        it.emit("B")
        it.emit("C")
    }
    
    f { print(it) } // ABC
    f { print(it) } // ABC
}
```

리시버 타입의 람다는 특정 객체(`FlowCollector`)위에서 직접 멤버 함수나 프로퍼티를 호출할 수 있게 해줍니다.  
즉, 람다식 내부에서 특정 객체의 참조 없이 해당 객체의 멤버를 직접 사용할 수 있게 됩니다.

이를 통해 `FlowCollector`의 `emit` 함수를 호출할 때, `FlowCollector`의 인스턴스 참조를 사용할 필요 없이 직접 `emit`을 호출할 수 있습니다.

```kotlin
fun interface FlowCollector {
    suspend fun emit(value: String)
}

suspend fun main() {
    val f: suspend FlowCollector.() -> Unit = {
        emit("A")
        emit("B")
        emit("C")
    }
    
    f { print(it) } // ABC
    f { print(it) } // ABC
}
```

람다식을 전달하는 대신, 인터페이스를 구현하는 객체를 통해 코드의 목적과 구조가 명확해지길 원할 수 있습니다.  
아래와 같이 인터페이스 `Flow`를 정의하고 이를 통해 구현하는 객체 내에서 원하는 모든 동작과 기능을 정의할 수 있습니다.

Kotlin에서 제공하는 객체 표현식을 사용하면, 인터페이스를 바로 구현하는 익명의 객체를 즉석에서 생성할 수 있습니다.  
이렇게 하면, 별도의 클래스 선언 없이도 인터페이스를 구현하고 해당 객체를 사용하는 것이 가능합니다.

```kotlin
fun interface FlowCollector {
    suspend fun emit(value: String)
}

interface Flow {
    suspend fun collect(collector: FlowCollector)
}

suspend fun main() {
    val builder: suspend FlowCollector.() -> Unit = {
        emit("A")
        emit("B")
        emit("C")
    }
    
    val flow: Flow = object: Flow {
        override suspend fun collect(collector: FlowCollector) {
            collector.builder()
        }
    }
    
    flow.collect { print(it) } // ABC
    flow.collect { print(it) } // ABC
}
```

`flow` 생성을 간소화하기 위해 `flow` 빌더를 정의할 수 있습니다.  
또한 `String`을 제네릭 타입으로 교체함으로 어떠한 타입의 값도 방출하고 수집할 수 있게 해줄 수 있습니다.

```kotlin
fun interface FlowCollector<T> {
    suspend fun emit(value: T)
}

interface Flow<T> {
    suspend fun collect(collector: FlowCollector<T>)
}

fun <T> flow(builder: suspend FlowCollector<T>.() -> Unit) = object : Flow<T> {
    override suspend fun collect(collector: FlowCollector<T>) {
        collector.builder()
    }
}

suspend fun main() {
    val flow: Flow<String> = flow {
        emit("A")
        emit("B")
        emit("C")
    }

    flow.collect { print(it) } // ABC
    flow.collect { print(it) } // ABC
}
```

---------------------------------------------------------------------------

## How Flow processing works

Flow는 리시버가 있는 중단 람다식보다 조금 더 복잡할 수 있지만, Flow의 강력함은 생성, 처리, 관찰을 위해 정의된 모든 함수들에 있습니다.  
이 모든것의 기본은 `flow`, `collect`, `emit`과 같은 기본적인 구성요소들에 있습니다. 

`map` 함수는 `Flow`의 각 요소를 변환하는 역할을 하며 다음과 같이 실행됩니다.

1. `map`은 `flow` 빌더를 사용하여 새로운 `Flow` 객체를 반환합니다.
2. 새로운 Flow 객체 생성 과정 중, `map` 함수는 원본 `Flow`의 `collect`를 호출하여 원본 `Flow`의 각 요소에 접근합니다.
3. 원본 `Flow`에서 요소를 하나씩 수집할 때마다, `map`은 파라미터 `transformation`을 사용하여 해당 요소를 변환합니다
4. 변환된 요소는 새로운 `Flow`에 `emit`을 통해 방출됩니다.

```kotlin
fun <T, R> Flow<T>.map(
    transformation: suspend (T) -> R
): Flow<R> = flow {
    this@map.collect { this.emit(transformation(it)) }
}

suspend fun main() {
    flowOf("A", "B", "C")
        .map {
            delay(1000)
            it.lowercase()
        }
        .collect { println(it) }
}
// 1s delay a
// 1s delay b
// 1s delay c
```

그 외의 유사한 함수들 입니다.

```kotlin
fun <T> FLow<T>.filter(
    predicate: suspend (T) -> Boolean
): Flow<T> = flow {
    collect {
        if (predicate(it)) emit(it)
    }
}

fun <T> Flow<T>.onEach(
    action: suspend (T) -> Unit
): Flow<T> = flow {
    collect {
        action(it)
        emit(it)
    }
}

fun <T> Flow<T>.onStart(
    action: suspend () -> Unit
): Flow<T> = flow {
    action()
    collect { emit(it) }
}
```

---------------------------------------------------------------------------

## Flow is synchronous

`Flow`의 동기적 특성은 데이터 스트림 처리에 중요한 특징입니다.  
`Flow`를 사용할 때, 각 단계의 실행은 순차적으로 이루어지며, 특정 단계가 완료되기 전에 다음 단계로 넘어가지 않습니다.

`collect`는 `Flow`의 모든 요소를 수집하는 동안 중단되며, 이는 `Flow`가 비동기적으로 동작하지 않고, 새로운 코루틴을 생성하지 않음을 나타냅니다.  
그러나 `Flow`의 각 처리 단계는 코루틴을 생성할 수 있는 중단 함수(`map`, `filter` 등)를 사용할 수 있음을 알아야 합니다.

또한 `onEach`에서 `delay`를 사용할 때, `Flow`의 동기적 특성으로 인해 `delay`는 각 요소 사이에 적용됩니다.

```kotlin
suspend fun main() {
    flowOf("A", "B", "C")
        .onEach { delay(1000) }
        .collect { println(it) }
}
// 1s delay A
// 1s delay B
// 1s delay C
```

---------------------------------------------------------------------------

## Flow and shared states

`Flow`는 동기적 특성을 지니기에, 한 번에 하나의 연산을 실행합니다.  
따라서 `Flow` 내부의 특정 단계에서 상태를 수정하는 것은 안전하다고 할 수 있습니다.

예를 들어 아래 예제와 같이 `Flow` 연산 내 로컬 변수를 사용하여 일부 데이터를 임시로 저장하거나 연산의 중간 결과를 계산하는 것은 안전합니다.  
이러한 변수는 해당 `Flow` 단계 내에서만 접근되며 다른 코루틴이나 스레드로부터 동시에 접근되지 않습니다.

그러나 여러 코루틴에서 동시에 실행되는 `Flow` 혹은 다른 코루틴과 공유되는 상태에 대해서는 주의가 필요합니다.  
이러한 경우 동시성 문제가 발생될 수 있으므로 동기화 메커니즘을 사용하여 상태를 보호해야 합니다.

```kotlin
fun <T, K> Flow<T>.distinctBy(
    keySelector: (T) -> K
) = flow {
    val sentKeys = mutableSetOf<K>()
    collect { value ->
        val key = keySelector(value)
        
        if (key !in sentKeys) {
            sentKeys.add(key)
            emit(value)
        }
    }
}
```

다음은 `flow` 단계 내에서 `counter` 변수를 사용하여 항상 1000까지 증가되는 일관된 예제 입니다.

```kotlin
fun Flow<*>.counter() = flow<Int> {
    var counter = 0
    collect {
        counter++
        List(100) { Random.nextLong() }.shuffled().sorted()
        emit(counter)
    }
}

suspend fun main(): Unit = coroutineScope {
    val f1 = List(1000) { "$it" }.asFlow()
    val f2 = List(1000) { "$it" }.asFlow().counter()

    launch { println(f1.counter().last()) } // 1000
    launch { println(f1.counter().last()) } // 1000
    launch { println(f2.last()) } // 1000
    launch { println(f2.last()) } // 1000
}
```

`Flow` 단계 외부에서 변수를 함수로 가져오게되면, 해당 변수는 동일한 `flow`에서 데이터를 수집하는 모든 코루틴에 의해 공유됩니다.  
이렇게 되면 예상치 못한 결과나 병렬 처리 문제가 발생될 수 있습니다.

아래 예시에서 `f2.last()`가 2000을 반환하는데, 이것은 2개의 `flow` 실행이 동시에 일어나면서 각각의 `flow`에서 `counter` 작업이 병렬로 이루어지기 떄문입니다.
만약 해당 변수가 `flow-specific`가 아니라면 이러한 문제가 발생될 수 있습니다.

따라서 `Flow`와 코루틴을 사용할 때는 변수의 범위와 사용방법에 대해 신중해야 합니다.

```kotlin
fun Flow<*>.counter(): Flow<Int> {
    var counter = 0
    return this.map {
        counter++
        List(100) { Random.nextLong() }.shuffled().sorted()
        counter
    }
}

suspend fun main(): Unit = coroutineScope {
    val f1 = List(1000) { "$it" }.asFlow()
    val f2 = List(1000) { "$it" }.asFlow().counter()

    launch { println(f1.counter().last()) } // 1000
    launch { println(f1.counter().last()) } // 1000
    launch { println(f2.last()) } // 2000
    launch { println(f2.last()) } // 2000
}
```

동일한 변수를 사용하는 중단 함수가 동기화가 필요한 것처럼, 함수 외부에서 정의되거나 클래스의 범위 또는 최상위 수준에서 정의된 `flow` 내의 변수도 동기화가 필요합니다.

```kotlin
var counter = 0

fun Flow<*>.counter(): Flow<Int> = map {
    counter++
    List(100) { Random.nextLong() }.shuffled().sorted()
    counter
}

suspend fun main() = coroutineScope {
    val f1 = List(1000) { "$it" }.asFlow()
    val f2 = List(1000) { "$it" }.asFlow().counter()

    launch { println(f1.counter().last()) } // 4000
    launch { println(f1.counter().last()) } // 4000
    launch { println(f2.last()) } // 4000
    launch { println(f2.last()) } // 4000
}
```