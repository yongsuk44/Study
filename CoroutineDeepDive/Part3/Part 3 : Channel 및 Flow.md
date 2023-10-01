# Channel and Flow

## 목차

- [Channel](#part-31--channel)
- [Select](#part-32--select)
- [Hot and cold data sources](#part-33--hot-and-cold-data-sources)
- [Flow introduction](#part-34--flow-introduction)
- [Understanding flow](#part-35--understanding-flow)
- [Flow building](#part-36--flow-building)
- [Flow lifecycle functions](#part-37--flow-lifecycle-functions)

## [Part 3.1 : Channel](Channel.md)

`Channel`은 여러 코루틴이 동시에 데이터를 보내거나 받을 수 있도록 설계되어 있으며 이 `Channel`을 통해 데이터를 주고 받을 수 있습니다.  
단, `Channel`로 보내진 데이터는 오직 1번만 수신할 수 있습니다. 이는 데이터의 일관성과 무결성을 보장하는데 중요합니다.

`Channel`은 `SendChannel`과 `ReceiveChannel`를 구현하며 다음과 같은 특징을 지닙니다.

- SendChannel : 데이터를 `Channel`에 보내기 위한 메서드로 `Channel`을 통해 수신자에게 데이터를 제공할 수 있습니다.  
  또한 `Channel`을 닫을 수 있어 수신자가 더 이상 데이터를 받지 못하도록 할 수 있습니다.
- ReceiveChannel : `Channel`로부터 데이터를 받기 위한 메서드로 `Channel`을 통해 데이터를 수신할 수 있습니다.

각 인터페이스는 `send()`와 `receive()` suspending 함수를 지원하며 이를 통해 동시성 제어를 할 수 있습니다.

- `send()` : `Channel`이 가득 차 있으면, `send()`는 `Channel`이 비워질 때까지 일시 중단됩니다.
- `receive()` : `Channel`이 비어 있으면, `receive()`는 `Channel`이 채워질 때까지 일시 중단됩니다.

만약 suspending 함수를 사용하지 않는 곳에서 데이터 송신과 수신을 해야하는 경우 `trySend`와 `tryReceive` 함수를 사용하여 즉시 성공과 즉시 실패를 반환받을 수 있습니다.

`Channel`에서 데이터 수신 중 수신자가 얼마나 많은 데이터를 수신해야 하는지 모를 수 있습니다.  
이러한 문제를 `Channel`이 닫힐 때까지 요소를 계속 수신하는 방법으로 `for-loop`와 `consumeEach`를 사용하여 해결할 수 있습니다.  
또한 더 간단한 방법으로 `produce` 코루틴 빌더를 사용할 수 있습니다.

`produce`는 `Channel`을 생성하고 `Channel`에 데이터를 보내는 코루틴을 생성합니다.  
또한 코루틴 본문에서 예외가 발생한 경우 자동으로 `Channel`을 닫아 안전하게 리소스를 정리하고
다른 코루틴에게 더 이상 데이터가 없음을 알려 안전하게 `Channel`을 관리할 수 있습니다.

---

### Channel types

채널의 타입은 버퍼 용량을 기반으로 구분할 수 있습니다.

- UNLIMITED : 버퍼 용량이 무한한 채널로, `send()`는 항상 일시 중단되지 않습니다.
- BUFFERED : 정해진 크기(기본 값 64)의 버퍼 용량을 가진 채널로, `send()`는 버퍼가 가득 차면 일시 중단됩니다.
- RENDEZVOUS : 버퍼 용량이 0인 채널로, `send()`와 `receive()`가 모두 준비되어야 데이터 전송을 할 수 있습니다.
- CONFLATED : 버퍼 용량이 1인 채널로, 버퍼에 항상 마자막으로 전송된 아이템만 유지되며 새로운 아이템이 들어오면 이전 아이템은 새로운 아이템으로 대체 됩니다.

---

### On buffer overflow

`onBufferOverflow` 파라미터는 `Channel`의 버퍼가 가득 찬 경우, 동작을 제어하여 `Channel`의 동작을 더욱 세밀하게 제어할 수 있습니다.

- SUSPEND(기본) : `send()`에서 코루틴을 중단시킵니다.
- DROP_OLDEST : 버퍼의 가장 오래된 데이터를 삭제합니다.
- DROP_LATEST : 버퍼의 가장 최근 데이터를 삭제합니다.

---

### On undelivered element handler

`onUndeliveredElement`는 `Channel`의 생명주기와 관련된 문제나 다양한 상황에서 예외가 발생했을 때 이 핸들러를 활용할 수 있습니다.

예로 파일을 전송하는 `Channel`에서 오류가 발생하여 리소스를 닫지 못하면 리소스 누수 문제가 발생하게 됩니다.  
이 떄, `onUndeliveredElement`를 사용하여 리소스를 정리하는 작업을 할 수 있습니다.

따라서 리소스 관리가 중요한 `Channel`에서 `onUndeliveredElement`를 활용하여 예기치 못한 상황에 대비합니다.

--- 

### Fan-out

- Fan-out : 여러 코루틴이 하나의 `Channel`에서 데이터를 수신하는 패턴을 의미합니다.

하나의 `Channel`에서 여러 코루틴이 안전하게 데이터를 수신하기 위해서는 `consumeEach` 보다는 `for-loop`를 통해 데이터를 수신하는 것이 안전합니다.

이와 같이 'Fan-out' 패턴은 `Channel`이 데이터나 요소를 `Queue` 방식으로 처리하며, 첫 번째로 대기하던 코루틴이 먼저 데이터를 수신합니다.

--- 

### Fan-in

- Fan-in : 여러 코루틴에서 하나의 `Channel`로 데이터를 보내는 패턴을 의미합니다.

여러 코루틴에서 동시에 작업을 처리하고 그 결과를 하나의 `Channel`로 집중시킬 때 유용합니다.
주의할 점으로는 여러 코루틴에서 동시에 데이터를 전송하면 데이터 순서가 보장되지 않기에 별도의 처리가 필요할 수 있습니다.

여러 `Channel`을 하나로 병합하려면 `produce`를 통해 `Channel`을 생성하고 `Channel`을 병합하는 방식으로 구현할 수 있습니다.

---

### Pipelines

일련의 데이터 처리 단계를 나타내는 용어로, 코루틴에서는 하나의 `Channel`에서 데이터를 받고 가공하여 다른 `Channel`로 전달되는 구조를 의미합니다.

---

### Summary

- `Channel`은 여러 코루틴이 동시에 데이터를 보내거나 받을 수 있도록 설계되어 데이터를 주고 받을 수 있습니다.  
  단, `Channel`로 보내진 데이터는 데이터의 일관성과 무결성으로 인해 오직 1번만 수신할 수 있습니다.
- `Channel`은 `SendChannel`과 `ReceiveChannel`을 구현하여 데이터를 보내고 받을 수 있습니다.
- 각 `Send`와 `Receive`는 suspending 함수를 지원하며 이를 통해 동시성 제어를 할 수 있습니다.  
  즉, `Channel`이 비어있거나 가득 찬 경우 코루틴이 일시 중지될 수 있습니다.
- `produce`는 `Channel`을 생성하고 생성된 `Channel`에 데이터를 보내는 코루틴을 생성합니다.  
  또한 예외가 발생한 경우 `Channel`을 닫아 리소스 정리하여 안전하게 관리할 수 있습니다.
- `Channel`의 타입은 버퍼 용량을 기반으로 구분하며 각 `UNLIMITED`, `BUFFERED`, `RENDEZVOUS`, `CONFLATED`로 구분할 수 있습니다.
- `onBufferOverflow` 파라미터는 `Channel`의 버퍼가 가득 찬 경우 동작을 제어하며, `SUSPEND`, `DROP_OLDEST`, `DROP_LATEST` 등 각 타입으로 지원합니다.
- `onUndeliveredElement`는 `Channel`의 생명주기와 관련된 문제나 다양한 상황에서 예외가 발생했을 때 이 핸들러를 활용할 수 있습니다.
- 'Fan-out' : 여러 코루틴이 하나의 `Channel`에서 데이터를 수신하는 패턴을 의미합니다.
- 'Fan-in' : 여러 코루틴에서 하나의 `Channel`로 데이터를 보내는 패턴을 의미합니다.

------------------------------------------------------------------------------------------------

## [Part 3.2 : Select](Select.md)

코루틴의 `select`는 여러 코루틴 중 먼저 완료되는 코루틴을 기다리는 제공을 하여 이를 통해 여러 작업 중 빠르게 처리되는 작업의 결과를 얻을 수 있습니다.
추가로 여러 채널 중 데이터를 전송하거나, 데이터를 수신할 수 있는 첫 번째 채널을 선택하는 것도 가능합니다.

### Selecting deferred values

코루틴에서 대표적인 비동기 처리 방법으로는 `async`를 통해 처리하는 방법이 있습니다.

이러한 비동기 작업을 여러 작업을 동시에 실행하여 가장 먼저 처리되는 작업의 결과를 얻고 싶은 상황이 발생될 수 있습니다.
이러한 경우 `select`와 `async`를 같이 사용하여 가장 먼저 완료되는 비동기 작업을 얻을 수 있습니다.

주의할 점으로 하나의 코루틴 스코프에서 `select`를 통해 여러 비동기 작업을 실행할 때,   
코루틴의 [구조적 동시성 메커니즘](../Structured%20Concurrency.md)에 의해 모든 비동기 작업이 처리되지 않으면 해당 스코프가 완료되지 않는 비효율적인 처리가 될 수 있습니다.

이러한 문제를 해결하기 위해 `select`로 결과를 선택한 후 `also`를 사용하여 완료되지 않은 비동기 작업을 취소하는 별도의 작업을 추가하는 것이 효율적입니다.

```kotlin
select<User> {
    async { getRestApi1() }.onAwait { it }
    async { getRestApi2() }.onAwait { it }
}.also { coroutineContext.cancelChildren() }
```

---

### Selecting from channels

`select`과 채널을 같이 사용하여 여러 채널 중 데이터를 전송하거나, 데이터를 수신할 수 있는 첫 번째 채널을 선택할 수 있으며 다음 함수들을 지원합니다.

- onReceive : 채널에 데이터가 있을 떄 데이터를 받아올 수 있으며 `select`는 람다식의 결과를 반환합니다.
- onReceiveCatching : 채널에 데이터가 있을 떄 데이터를 받아올 수 있고, 추가로 채널이 닫혔을 떄 채널을 정리하는 등의 처리할 수 있습니다.
  `select`는 람다식의 결과를 반환합니다.
- onSend : 채널의 버퍼에 공간이 있을 때 데이터를 전송할 수 있습니다. `select`는 `Unit`을 반환합니다.

--------------------------------------------------------------------

## [Part 3.3 : Hot and cold data sources](Hot%20and%20cold%20data%20sources.md)

### Hot vs Cold

Hot Stream은 소비자의 구독과 관계 없이 데이터를 계속해서 생성하며, 생성된 데이터를 저장하고 있다가 필요할 때 제공할 수 있습니다.

Cold Stream은 데이터가 필요할 때까지 어떠한 연산도 수행하지 않습니다. 즉, 구독자가 데이터를 요청할 때만 데이터를 생성하고 연산을 수행하여 제공합니다.

이러한 특징으로 Cold Stream은 요청이 올 때 데이터를 생성하므로 이론적으로 데이터 길이가 무한할 수 있습니다.  
또한 최소한의 연산만을 수행하기에 연산의 최적화와 효율성을 가지며, 중간 결과를 저장하는 컬렉션이 없기에 메모리 사용량이 적습니다.

### Hot channels, cold flow

`Channel`과 `flow` 빌더는 개념적으로 유사하지만, 동작 방식에 차이가 있습니다.

`Channel`은 Hot Stream으로 즉시 값을 계산하여 시작합니다.  
이러한 계산은 수신자가 준비될 때까지 중지되며, `Channel`은 소비와 무관하게 요소를 독립적으로 생산하고 보관합니다.
또한 요소는 1번만 수신될 수 있기에 하나의 수신자가 모든 요소를 수신하면, 다른 수신자는 아무런 요소를 받을 수 없습니다.

`Flow`는 Cold Stream으로 요청에 따라 요소가 생산됩니다.  
또한 `Flow`는 처리 작업을 하지 않는 단순한 정의로 터미널 연산(`collect`)가 수행될 때 요소가 어떻게 생산될지에 대해 정의합니다.
즉, 터미널 연산을 여러번 시도하면 계속해서 요소를 사용할 수 있습니다.

------------------------------------------------------------------

## [Part 3.4 : Flow introduction](Flow%20introduction.md)

### The characteristics of Flow

`collect`와 같은 터미널 연산은 스레드 차단이 아닌 코루틴을 중지하며, 현재 코루틴 컨텍스트를 존중하여 다른 코루틴 컨텍스트들의
기능(`CoroutineExceptionHandler`, `CoroutineName` 등)들을 지원합니다.  
또한 구조적 동시성을 지원하기에 부모 코루틴이 취소되면, 내부에서 실행 중인 Flow 연산도 함께 취소됩니다.

`flow` 빌더는 일시 중지 함수가 아니며, 특별한 코루틴 없이 사용할 수 있지만, 터미널 연산은 코루틴을 중지한다는 특징을 알아야 합니다.

### Flow nomenclature

모든 Flow는 시작점이 필요하며 `flow` 빌더, 다른 객체로부터의 변환, 헬퍼 함수로 시작할 수 있습니다.  
터미널 연산은 Flow 처리를 마무리하는 단계로, 일시 중지 될 수 있습니다.  
Flow 시작과 터미녈 연산 사이에 중간 연산을 통해 데이터를 변형하거나 필터링 하는 등의 작업을 할 수 있습니다.

### Real-life use cases

다음은 Flow의 일반적인 사용 사례입니다.

- 웹소켓, 알림 등 서버와 메시지를 계속해서 주고받을 때
- 버튼 클릭, 텍스트 입력과 같은 연속적인 데이터 스트림으로 간주되는 사용자 이벤트 관찰
- GPS, 가속도계 등 센서에서 수신되는 연속적인 데이터 정보
- 데이터 베이스가 변화되는것을 관찰

또한 많은 양의 API를 요청하는 상황에서 `flatMapMerge` 중간 연산의 `concurrency` 파라미터를 추가하여 호출의 수를 제한하는 방법을 제공합니다.

------------------------------------------------------------------

## [Part 3.5 : Understanding flow](Flow%20이해하기.md)

Flow는 여러 여산들의 실행 정의를 나타내는 개념으로 중단 람다식과 유사합니다.

중단 람다식은 특정 연산이나 작업을 잠시 중단하고 나중에 재개 해주는 기능을 가진 코드 블록을 의미합니다.  
이는 기본 람다식에 `suspend` 키워드를 사용하여 만들 수 있으며, 코루틴 내에서 `delay`와 같이 일시적으로 정지할 수 있습니다.

그러나 이러한 람다식들은 코드 복잡성을 증가시킬 수 있기에, 이러한 람다 구현을 함수형 인터페이스로 정의하여 인스턴스를 전달할 필요 없이 직접 람다식을 사용하여 호출할 수 있습니다.

또한 함수형 인터페이스는 람다식으로 표현될 수 있기에 해당 람다식을 리시버 타입으로 만들어 특정 객체 참조 없이 멤버를 직접 사용할 수 있도록 할 수 있습니다.

```kotlin
fun interface FlowCollector<T> {
    suspend fun emit(value: T)
}

val f: suspend FlowCollector<String>.() -> Unit = {
    emit("A")
}
```

그럼에도 람다식 대신 인터페이스를 통해 코드의 목적과 구조가 명확해지길 원한다면 다음과 같이 정의할 수 있습니다.

```kotlin
interface Flow<T> {
    suspend fun collect(collector: FlowCollector<T>)
}

val builder: suspend FlowCollector<String>.() -> Unit = {
    emit("A")
}

val flow: Flow<String> = object : Flow<String> {
    override suspend fun collect(collector: FlowCollector<String>) {
        collector.builder()
    }
}
```

한번 더 `flow`의 생성을 간소화하기 위해 `flow` 빌더를 정의할 수 있습니다.

```kotlin
fun <T> flow(builder: suspend FlowCollector<T>.() -> Unit) = object : Flow<T> {
    override suspend fun collect(collector: FlowCollector<T>) {
        collector.builder()
    }
}

val flow: Flow<String> = flow {
    emit("A")
}
```

---

### Flow is synchronous

Flow는 동기적 특성을 가집니다. 즉, 각 단계의 실행은 순차적으로 이루어지고 특정 단계가 완료되기 전 다음 단계로 넘어가지 않습니다.

예를 들어 `collect`는 모든 요소를 수집하는 동안 중단되며, 이는 `Flow`가 동기적이고 새로운 코루틴을 생성하지 않음을 나타냅니다.
또한 `onEach`에서 `delay`를 사용하면 동기적 특성으로 인해 각 요소 사이에 `delay`가 적용됩니다.

---

### Flow and shared states

`Flow`는 동기적 특징을 지니기에, `Flow` 연산 내 로컬 변수를 사용하여 일부 데이터를 임시로 저장하거나,
연산의 중간 결과를 계산하는 등의 작업은 안전하다고 할 수 있습니다.

그러나 여러 코루틴에서 동시에 실행되는 `Flow` 혹은 다른 코루틴과 공유되는 상태를 사용할 때는
동시성 문제가 발생될 수 있으므로 동기화 메커니즘을 사용하여 공유된 상태를 보호해야 합니다.

------------------------------------------------------------------

## [Part 3.6 : Flow building](Flow%20Building.md)

### Flow from raw values

`Flow`를 생성하는 가장 간단한 방법은 `flowOf` 함수를 사용하는 것입니다.
또한 비어있는 `Flow`를 생성하려면 `emptyFlow`를 사용할 수 있습니다.

```kotlin
flowOf(1, 2, 3, 4, 5).collect()
emptyFlow<Int>().collect()
```

--- 

### Converters

`asFlow`는 `Iterable`, `Iterator`, `Sequence`와 같은 컬렉션 형태의 데이터 구조를 `Flow`로 변환하는 데 사용됩니다.

```kotlin
listOf(1, 2, 3, 4, 5)
    // setOf(1, 2, 3, 4, 5)
    // sequenceOf(1, 2, 3, 4, 5)
    .asFlow()
    .collect()
```

---

### Converting a function to a flow

기존의 suspending 함수나 일반 함수의 결과를 `Flow`로 변환하는 경우, `Flow`는 해당 함수의 결과만을 포함하게 됩니다.
즉, `suspend () -> T` or `() -> T`의 경우 `Flow<T>`로 변환됨을 말합니다.

또한 참조 연산자(`::`)를 통해 특정 함수를 참조하여 `Flow`로 변환할 수 있습니다.

---

### Flow builders

`flow { ... }` 빌더는 `Flow`를 생성하는 일반적인 방법으로 순차적 데이터 구조 혹은 채널을 생성하는 방식과 유사하게 동작합니다.

`emit`을 통해 원하는 값을 `Flow`에 추가할 수 있으며 `emitAll`을 사용하면 채널이나 다른 `Flow`의 값을 현재의 `Flow`에 순차적으로 추가할 수 있습니다.

---

### Understanding flow builder

`flow` 빌더 호출 시 실제 코드에서는 `Flow` 객체를 생성하기만 합니다.  
생성된 `Flow` 객체는 내부적으로 `Flow` 인터페이스를 구현하며, `collect`를 호출하면 `flow` 빌더 내 정의된 코드 블록을 실행합니다.

코드 블록 내부에서는 `FlowCollector`를 통해 `emit`을 호출하여 `Flow`에 값을 추가합니다.  
이 후에 `collect`에 원하는 값을 전달하여 마무리 합니다.

이러한 방식으로 `Flow`는 직관적이고 간단합니다.   
`Flow`에 대한 다양한 확장 함수나 추가 기능들은 모두 위 원리를 기반으로 구축됩니다.

---

### ChannelFlow

`channelFlow`는 `Flow`와 같이 터미널 연산으로 시작되지만, `Channel`과 같이 독립적으로 데이터를 생성할 수 있습니다.
이렇게 생성된 데이터는 별도의 코루틴에서 처리되어 데이터의 요청과 처리가 동시에 이루어질 수 있습니다.  
즉, `Flow`와 `Channel`의 특징을 결합한 `Flow` 입니다.

`channelFlow`는 내부에서 `ProducerScope<T>` 작업을 수행하며,
`ProducerScope`는 `CoroutineScope`를 구현하기에 코루틴 생성, 코루틴 생명주기 관리, 다양한 코루틴 연산을 수행할 수 있습니다.

`ProducerScope`는 `SendChannel`을 구현하고 있어, `SendChannel`의 다양한 함수를 통해 채널을 닫거나, 특정 조건에 데이터 전송을 일시 정지하는 등의 채널 동작을 제어할 수 있습니다.
또한 내부에서 데이터를 생성하고 전송하는데 `emit`이 아닌 `send`를 사용합니다.

---

### CallbackFlow

`callbackFlow`와 `channelFlow` 모두 독립적인 코루틴 환경에서 데이터 생성 및 전달을 수행합니다.  
그러나 `callbackFlow`는 콜백 기반 코드와의 통합을 목적으로 설계되었으며,
UI 이벤트와 네트워크 응답 등 비동기적인 콜백을 `Flow`로 표현할 때 사용합니다.

`callbackFlow` 다음 함수들을 통해 콜백 기안 비동기 작업을 `Flow`로 변환하고 관리할 수 있습니다.

- `awaitClose` : 코루틴이 콜백 등록 후 즉시 종료되는 것을 방지하며, 채널이 다른 방법으로 닫힐 때까지 코루틴을 일시 중지 상태로 유지합니다.
- `trySendBlocking` : 코루틴 일시 중지 없이 값을 채널에 전송하며, 일반 함수에서도 사용 가능합니다.
- `close` : 채널을 종료하고 해당 `Flow`의 데이터 전송을 중단합니다.
- `cancel` : 예외와 함께 채널을 종료하며 `Flow` 수신자에게 예외를 전달하여 오류 처리가 가능합니다.

------------------------------------------------------------------

## [Part 3.7 : Flow lifecycle functions](Flow%20lifecycle%20functions.md)

`Flow`는 다양한 이벤트를 감지하고 반응하는 기능을 제공합니다.

### onEach

`Flow` 데이터 스트림의 요소들을 순서대로 처리하며 일시 정지될 수 있어 `delay` 시 각 값을 지연하여 출력합니다.

```kotlin
flowOf(1,2)
    .onEach { delay(1000) }
    .collect { println(it) }
```