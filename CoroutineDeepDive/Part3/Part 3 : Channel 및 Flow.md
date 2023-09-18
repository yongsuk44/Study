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
