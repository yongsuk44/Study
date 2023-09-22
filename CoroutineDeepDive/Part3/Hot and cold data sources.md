# Hot and cold data sources

## Hot vs cold

### Hot Stream

1. 데이터를 미리 생성하고 저장하려는 경향이 있는데, 이는 데이터 생성의 주체와 소비의 주체가 분리되기 때문입니다.
2. 소비자의 구독 여부와 관계없이 데이터를 계속해서 생성합니다.
3. 생성된 데이터를 저장하고 있다가 필요할 때 제공할 수 있습니다.   
   예를 들어 `List`는 미리 생성된 요소들을 저장하고 있습니다.

### Cold Stream

1. 실제 데이터가 필요할 때까지 어떠한 연산도 수행하지 않습니다.
2. 구독자가 데이터를 요청할 때만 해당 데이터를 생성하고 연산을 수행합니다.
3. 생성된 데이터를 저장하지 않습니다. 대신 필요할 때마다 다시 생성하거나 연산을 수행합니다.  
   예를 들어 `Sequence`는 요청에 따라 데이터를 생성합니다.

```kotlin
fun main() {
    val l = buildList {
        repeat(3) {
            add("User$it")
            println("L: Added User")
        }
    }

    val l2 = l.map {
        println("L: Processing")
        "Processed $it"
    }

    val s = sequence {
        repeat(3) {
            yield("User$it")
            println("S: Added User")
        }
    }

    val s2 = s.map {
        println("S: Processing")
        "Processed $it"
    }
}

// L: Added User
// L: Added User
// L: Added User
// L: Processing
// L: Processing
// L: Processing
```

결과적으로 Cold 스트림은 다음과 같은 특징을 지님을 알 수 있습니다.

- 무한할 수 있습니다. 즉, 요청을 받을 때 마다 데이터를 생성하므로 이론적으로 무한한 길이의 데이터를 생성할 수 있습니다.
- 최소한의 연산만을 수행합니다. 실제로 데이터가 필요한 시점에만 연산을 수행합니다. 이는 연산의 최적화 및 효율성을 의미합니다.
- 더 적은 메모리를 사용합니다. 중간 결과를 저장하는 컬렉션을 만들 필요가 없기에 메모리 사용량이 줄어듭니다.

--------------------------------------------------------------------------------------------

## Hot channels, cold flow

`flow`는 `produce` 함수와 유사한 방식으로 동작하며 `flow` 블록 내부에서 데이터를 생성하고 전송할 수 있습니다.

```kotlin
val channel = produce {
    while (true) {
        val x = computeNextValue()
       send(x)
    }
}

val flow = flow {
    while (true) {
        val x = computeNextValue()
        emit(x)
    }
}
```

`Channel`과 `flow` 빌더는 개념적으로 유사하지만 그들의 동작 방식의 차이로 인해 두 함수 사이에 중요한 차이점이 있습니다.

`Channel`은 핫 스트림으로 코루틴에서 즉시 값을 계산하여 시작합니다.

이 계산은 수신자가 준비될 떄까지 중단되며, 채널은 소비와 무관하게 요소를 독립적으로 생산하고 보관합니다.  
요소는 1번만 수신될 수 있기에, 하나의 수신자가 모든 요소를 소비하면 다른 수신자는 비어 있고 이미 닫힌 채널을 찾게 됩니다.

```kotlin
private fun CoroutineScope.makeChannel() = produce {
    println("Channel started")

    for (i in 1..3) {
        delay(1000)
        send(i)
    }
}

suspend fun main() = coroutineScope {
    val channel = makeChannel()

    delay(1000)

    println("Calling channel...")
    for (value in channel) println(value)

    println("Consuming again...")
    for (value in channel) println(value)
}

// Channel started
// 1s delay
// Calling channel...
// 1
// 1s delay
// 2
// 1s delay
// 3
// Consuming again...
```

`Flow`는 콜드 스트림으로 요청에 따라 요소가 생산됩니다.  
또한 `Flow`는 처리 작업을 하지 않는 단순한 정의로 터미널 연산(예: `collect`)이 수행될 때 요소가 어떻게 생성될지에 대해 정의합니다.

이러한 특성으로 `flow` 빌더는 `CoroutineScope`가 필요하지 않으며 모든 터미널 연산은 처음부터 처리를 시작합니다.

```kotlin
private fun makeFlow() = flow {
    println("Flow started")

    for (i in 1..3) {
        delay(1000)
        emit(i)
    }
}

suspend fun main() = coroutineScope {
    val flow = makeFlow()

    delay(1000)

    println("Calling flow...")
    flow.collect { println(it) }

    println("Consuming again...")
    flow.collect { println(it) }
}
// 1s delay
// Calling flow...
// Flow started
// 1s delay
// 1
// 1s delay
// 2
// 1s delay
// 3
// Consuming again...
// Flow started
// 1s delay
// 1
// 1s delay
// 2
// 1s delay
// 3
```