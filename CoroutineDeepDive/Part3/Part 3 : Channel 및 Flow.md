# Channel and Flow

## 목차

- [Channel](#part-31--channel)
- [Select](#part-32--select)
- [Hot and cold data sources](#part-33--hot-and-cold-data-sources)
- [Flow introduction](#part-34--flow-introduction)

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

`collect`와 같은 터미널 연산은 스레드 차단이 아닌 코루틴을 중지하며, 현재 코루틴 컨텍스트를 존중하여 다른 코루틴 컨텍스트들의 기능(`CoroutineExceptionHandler`, `CoroutineName` 등)들을 지원합니다.  
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