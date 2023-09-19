# Channel

`Channel API`는 코루틴 간 데이터를 전송하기 위해 사용되는 통신 도구입니다.  
일반적으로 채널이 데이터를 전달하는 파이프처럼 작동한다고 생각할 수 있습니다.

그러나 더 직관적인 비유로 도서관의 책장으로 생각할 수도 있습니다.  
책장에서는 한 사람이 책을 가져다 놓으면 다른 사람이 그 책을 가져갈 수 있습니다.

이런 방식으로 코루틴 내에서 어느 코루틴이 `Channel`에 데이터를 보내면, 다른 코루틴이 그 데이터를 받아올 수 있습니다.
이처럼 데이터를 주고 받는 두 코루틴 사이의 **직접적인 연결을 요구하지 않기에** 유용합니다.

---

`Channel`은 여러 개의 코루틴이 동시에 데이터를 보내거나 받을 수 있도록 설계되어 있습니다.  
이는 여러 코루틴이 동일한 `Channel`을 통해 정보를 주고 받을 수 있음을 의미합니다.

여기서 중요한 점은 `Channel`로 보내진 특정 값은 오직 **1번만 수신**될 수 있습니다.
이러한 특징은 `Channel`이 데이터의 일관성과 무결성을 보장하는데 중요합니다.

```mermaid
graph LR
    subgraph Channel#1
        direction LR
        coroutine#1 -- send --> channel#1 -- receive --> coroutine#2
    end

    subgraph LR Channel#2
        direction LR
        producer#1 -- send --> channel#2 -- receive --> consumer#1
        producer#2 -- send --> channel#2 -- receive --> consumer#2
        producer#3 -- send --> channel#2 -- receive --> consumer#3
    end
```

---

### Channel

```kotlin
interface Channel<E> : SendChannel<E>, ReceiveChannel<E>
```

`Channel` 인터페이스는 데이터를 보내는 작업과 데이터를 받는 작업을 위해 설계 되었습니다.  
이 2가지 작업을 명확히 구분하기 위해 `Channel`은 `SendChannel`과 `ReceiveChannel` 2개의 서브 인터페이스를 구현합니다.

### SendChannel

```kotlin
interface SendChannel<in E> {
    suspend fun send(element: E)
    fun close(): Boolean
    // ...
}
```

- 요소를 `Channel`에 보내기 위한 메서드들을 정의합니다.
- 코루틴이 데이터를 `SendChannel`을 통해 보내면, 이 데이터는 `Channel`의 수신자에게 제공될 수 있습니다.
- `SendChannel` 인터페이스를 통해 `Channel`을 닫을 수 있어 `Channel`의 수신자가 더 이상 데이터를 받지 못하도록 할 수 있습니다.

### ReceiveChannel

```kotlin
interface ReceiveChannel<out E> {
    suspend fun receive(): E
    fun cancel(cause: CancellationException? = null)
    // ...
}
```

- `Channel`에서 요소를 받기 위한 메서드들을 정의합니다.
- 코루틴이 `ReceiveChannel`을 통해 데이터를 받으면, 이 데이터는 `Channel`의 송신자에게 제공될 수 있습니다.

---

이처럼 `Channel`의 진입점을 제한하기 위해 `ReceiveChannel` 또는 `SendChannel`만 노출할 수 있습니다.

또한 각 인터페이스의 `send()`와 `receive()` 메서드들이 supsending 함수인것을 통해 다음과 같이 동시성 제어가 가능함을 짐작할 수 있습니다.

### 수신 시 일시 정지

코루틴이 데이터를 `Channel`에서 수신하려 시도할 때, 해당 데이터가 현재 `Channel`에 없으면 코루틴은 일시 정지됩니다.
즉, CPU 리소스를 낭비하지 않고 다른 작업을 수행할 수 있습니다. 데이터가 채널에 도착하면 코루틴은 자동으로 재개됩니다.

### 송신 시 일시 정지

`Channel`에 데이터를 전송하려 시도할 때, `Channel`의 용량이 가득 찼다면 해당 코루틴은 일시 정지됩니다.
이는 `Channel`의 용량이 초과되어 데이터 추가를 방지하며, 공간이 확보되면 코루틴은 자동으로 재개됩니다.

### trySend & tryReceive

만약, 일시 정지되지 않는 함수에서 데이터를 보내거나 받아야 할 경우, `trySend`와 `tryReceive`를 사용할 수 있습니다.
이 메서드 들은 `Channel`의 현재 상태에 따라 즉시 성공 또는 즉시 실패를 반환하므로 일시 정지되지 않습니다.

---

`Channel`은 여러 개의 송신자와 수신자를 가질 수 있음을 계속해서 설명했습니다.
그러나 실제 코루틴 기반 앱들은 `Channel`의 송신과 수신 측에 하나의 코루틴이 있는 경우가 일반적입니다.

이러한 구조는 두 코루틴 사이의 직접적인 데이터 교환을 가능하게 하며, 더 복잡한 동시성 구조를 간단하게 유지할 수 있습니다.

`Channel`는 별도의 코루틴에서 생산자(송신자, producer)와 소비자(수신자, consumer)를 가져야 합니다.
즉, 생산자는 요소를 보내고, 소비자는 그것을 받습니다. 아래는 `Channel`의 간단한 구현입니다.

```kotlin
suspend fun main() = coroutineScope {
    val channel = Channel<Int>()

    launch {
        repeat(5) { i ->
            delay(1000)
            println("Producing next one")
            channel.send(i * 2)
        }
    }

    launch {
        repeat(5) {
            val received = channel.receive()
            println(received)
        }
    }
}
// 1s delay
// Producing next one
// 0
// 1s delay
// Producing next one
// 2
// ...
// 1s delay
// Producing next one
// 8
```

그러나 위 구현은 완벽하다고 할 수 없습니다.
`Channel`에서 데이터를 수신하는 전형적인 문제 중 하나는 수신자가 얼마나 많은 데이터를 수신해야 하는지 모르는 것입니다.

이러한 문제를 해결하기 위해서 코루틴은 `Channel`이 닫힐 때까지 요소를 계속해서 수신하는 방법으로 `for-loop`나 `consumeEach`를 사용 할 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    val channel = Channel<Int>()

    launch {
        repeat(5) { i ->
            delay(1000)
            println("Producing next one")
            channel.send(i * 2)
        }
        channel.close()
    }

    launch {
        for (received in channel) {
            println(received)
        }

        // or 
        // channel.consumeEach { element ->
        //     println(element)
        // }
    }
}
```

위와 같은 방식은 `Channel`을 닫는 것을 잊어버리기 쉽습니다. 특히, 예외의 경우에 더욱 그렇습니다.
만약 코루틴이 예외로 인해 생성을 중단하면 다른 코루틴은 영원히 요소를 기다리게 되는 상황이 생길 수 있습니다.

이 때문에 코루틴은 `ReceiveChannel`을 반환하는 `produce` 코루틴 빌더를 제공합니다.

### Produce

`produce`는 `Channel`에서 데이터를 생성하는 코루틴을 생성할 때 사용되며 다른 코루틴에서 이 `Channel`을 통해 데이터를 수신 받을 수 있게 합니다.

또한 코루틴 본문에서 예외가 발생한 경우 자동으로 `Channel`을 닫아줍니다.

따라서 개발자가 별도로 `close` 호출을 잊더라도 안전하게 리소스를 정리하고 다른 코루틴에게 더 이상 데이터가 없음을 알릴 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    val channel = produce {
        repeat(5) { i ->
            delay(1000)
            println("Producing next one")
            send(i * 2)
        }
    }

    for (element in channel) println(element)
}
``` 

----------------------------------------------------------------------------

## Channel types

`Channel`의 동작 방식을 결정하는 중요한 파라미터인 용량을 기반으로 `Channel` 타입을 구분합니다.

#### 1. Unlimited

- 무제한 용량 버퍼를 가집니다.
- `send()` 호출 시 중단되지 않습니다. 즉, `Channel`에 데이터를 보내는 코루틴은 일시 정지되지 않습니다.

#### 2. Buffered

- 정해진 크기의 버퍼를 가집니다. 기본 값은 64이며, `kotlinx.coroutines.channels.defaultBuffer` 시스템 프로퍼티를 통해 재정의 할 수 있습니다.
- 버퍼가 가득 차면, 새로운 데이터를 전송하기 위해 송신자는 일시 정지 됩니다.

#### 3. Rendezvous (기본값)

- 용량이 0인 버퍼를 가집니다. 즉, 버퍼링을 하지 않습니다.
- 송신자는 수신자가 준비될 때까지 일시 정지됩니다. 이 때문에 송신자와 수신자가 동시에 활성화 되어있을 때만 데이터가 전송됩니다.

#### 4. Conflated

- 용량이 1인 버퍼를 가집니다. 즉, 버퍼에는 항상 마지막으로 전송된 아이템만 유지됩니다.
- 새로운 아이템이 전송되면, 이전 아이템은 새로운 아이템으로 대체됩니다.

코루틴의 `Channel`은 버퍼링 방식에 따라 다양한 동작을 함을 알 수 있습니다.

위에서 설명한것 처럼, `Channel.UNLIMITED`은 `send()`가 절대로 중단되지 않는 특징을 지닙니다.  
이는 생성자가 데이터를 가능한 빠르게 `Channel`에 보낼 수 있음을 의미하지만, 수신자가 데이터를 느리게 받더라도, `Channel`은 계속해서 보내진 데이터를 버퍼링하게 도비니다.

이렇게 버퍼링을 설정하면, 생산자는 데이터를 무제한으로 보낼 수 있으며, 수신자는 자신의 속도에 맞춰 하나씩 처리하게 됩니다.
이러한 특징은 생산자가 많은 양의 데이터를 빠르게 생성해야 하지만, 수신자는 이를 천천히 처리해야 하는 상황에서 사용될 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    val channel = produce(capacity = Channel.UNLIMITED) {
        repeat(5) { i ->
            send(i * 2)
            delay(100)
            println("Sent")
        }
    }

    delay(1000)

    for (element in channel) {
        println(element)
        delay(1000)
    }
}
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// (1 - 4 * 0.1) = 0.6s delay
// 0
// 1s delay
// 2
// 1s delay
// 4
// ...
// 8
// 1s delay
```

버퍼링된 `Channel`에는 지정된 용량만큼의 버퍼 공간이 있습니다. 생산자는 이 버퍼가 가득 찰 때까지 데이터를 보낼 수 있습니다.
버퍼가 가득 찬 상태에서 추가로 데이터를 보내려고 시도하면, `send()`는 수신자가 데이터를 수신하여 버퍼에 공간이 생길 때까지 중단됩니다.

```kotlin
suspend fun main() = coroutineScope {
    val channel = produce(capacity = 3) {
        repeat(5) { i ->
            send(i * 2)
            delay(100)
            println("Sent")
        }
    }

    delay(1000)
    for (element in channel) {
        println(element)
        delay(1000)
    }
}
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 1 - 2 * 0.1 = 0.8s delay
// 0
// Sent
// 0.1s delay
// 2
// Sent
// 1s delay
// 4
// 1s delay
// 6
// 1s delay
// 8
// 1s delay
```

`Channel.RENDEZVOUS`는 버퍼링을 하지 않습니다. 즉, 생산자와 수신자가 동시에 `send`와 `receive` 연산을 수행할 때만 데이터 전송이 가능합니다.

만약 생산자가 데이터를 보내려 시도하여도 해당 시점에 수신자가 데이터를 수신하려고 하지 않으면 생산자는 중단됩니다.
이는 반대로 수신자가 데이터를 수신하려 시도할 때 생산자가 데이터를 보내지 않으면 수신자가 중단됨을 의미합니다.

```kotlin
suspend fun main() = coroutineScope {
    val channel = produce { // or produce(capacity = Channel.REDENZVOUS) {
        repeat(5) { i ->
            send(i * 2)
            delay(100)
            println("Sent")
        }
    }

    delay(1000)
    for (element in channel) {
        println(element)
        delay(1000)
    }
}
// 0
// Sent
// 1s delay
// 2
// Sent
// 1s delay
// ...
// 8
// Sent
// 1s delay
```

`Channel.CONFLATED`를 사용하는 경우, 데이터를 보낼 때마다, 그 데이터는 이전에 버퍼에 저장된 데이터를 대체합니다.
따라서 수신자가 `Channel`로부터 데이터 수신 시, 항상 최근 데이터만 수신하게 됩니다.

```kotlin
suspend fun main() = coroutineScope {
    val channel = produce(capacity = Channel.CONFLATED) {
        repeat(5) { i ->
            send(i * 2)
            delay(100)
            println("Sent")
        }
    }

    delay(1000)
    for (element in channel) {
        println(element)
        delay(1000)
    }
}
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 1 - 4 * 0.1 = 0.6s delay
// 8
```

----------------------------------------------------------------------------

## On buffer overflow

`Channel`의 `onBufferOverflow` 파라미터는 버퍼가 가득 찼을 떄 동작을 제어하는 방식을 정의하여 `Channel`의 동작을 더욱 세밀하게 제어할 수 있게 해줍니다.

`onBufferOverflow`는 다음과 같은 옵션들이 있습니다.

- SUSPEND(기본값) : 버퍼가 가득 차면, `send()`에서 코루틴을 중단시킵니다.
- DROP_OLDEST : 버퍼가 가득 차면, 가장 오래된 요소를 삭제합니다.
- DROP_LATEST : 버퍼가 가득 차면, 가장 최근 요소를 삭제합니다.

추측할 수 있겠지만, `Channel.CONFLATED`를 사용하는 경우 `onBufferOverflow`를 `DROP_OLDEST`로 설정하는 것과 동일합니다.

그러나 현재 `produce`는 `onBufferOverflow`를 지원하지 않습니다.
따라서 이 설정을 사용하려면 `Channel`을 직접 호출하여 정의해야 합니다.

```kotlin
suspend fun main() = coroutineScope {
    val channel = Channel<Int>(
        capacity = 2,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )

    launch {
        repeat(5) { i ->
            channel.send(i * 2)
            delay(100)
            println("Sent")
        }
        channel.clsoe()
    }

    delay(1000)
    for (element in channel) {
        println(element)
        delay(1000)
    }
}
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 0.1s delay
// Sent
// 1 - 4 * 0.1 = 0.6s delay
// 6
// 1s delay
// 8
```

----------------------------------------------------------------------------

## On undelivered element handler

`onUndeliveredElement`는 `Channel`을 통해 전달된 요소가 제대로 처리되지 않았을 때 호출되는 함수입니다.
즉, `Channel`이 닫히거나 취소되었을 때를 의미하지만, `send`, `receive`, `receiveOrNull`, `hasNext`에서 예외가 발생할 수도 있습니다.

예를 들어 파일을 전송하는 `Channel`에서 파일 전송 중 오류가 발생되어 해당 파일의 리소스를 제대로 닫지 못했으면 리소스 누수와 같은 문제가 발생될 수 있습니다.
이런 상황에서 `onUndeliveredElement`를 사용하면 `Channel`로 전송되는 파일의 리소스를 안전하게 닫아 이러한 문제를 예방할 수 있습니다.

```kotlin
val channel = Channel<Resource>(capacity) { resource ->
    resource.close()
}
// or
// val channel = Channel<Resource>(
//    capacity = capacity,
//    onUndeliveredElement = { resource ->
//        resource.close()
//    }
// )

// Produce code
val resourceToSend = openResource()
channel.send(resourceToSend)

// Consumer code
val resourceReceived = channel.receive()
try {
    // use resourceReceived
} finally {
    resourceReceived.close()
}
```