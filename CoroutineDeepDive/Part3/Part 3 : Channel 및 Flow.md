# Channel and Flow

## 목차

- [Channel](#part-31--channel)

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

