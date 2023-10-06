# Kotlin Coroutines in practice

## 목차

- [Common use cases](#part-41--common-use-cases)

---

## [Part 4.1 : Common use cases](Common%20use%20cases.md)

대부분 Application 구조는 다음 3가지 계층으로 나눕니다.

- Data / Adapter Layer
- Domain Layer
- Presentation / UI Layer

### Data / Adapter Layer

데이터 저장, 추출, 변환 등 작업을 합니다.   
즉, 네트워크 요청, 데이터베이스 작업 등을 하는 영역입니다.

여기서 코루틴은 `suspend` 키워드를 통해 각 작업을 비동기적으로 수행할 수 있게 지원해줍니다.  
추가로, `Room` 라이브러리의 경우 `Flow`를 지원하며 쉬운 반응형 UI 구현을 도와줍니다.

#### callback functions

만약 코루틴을 지원하지 않고 콜백 함수를 사용하도록 강제하는 라이브러리를 사용해야 하는 경우,
`suspendCancellableCoroutine`을 사용하여 콜백 함수를 일시 중지 함수로 변환할 수 있습니다.

콜백 호출 시 `resume`을 통해 코루틴을 재개할 수 있으며, 콜백이 취소 가능한 경우 `invokeOnCancellation`을 통해 취소 로직을 정의할 수 있습니다.
또한 콜백 호출의 성공과 예외를 명확하게 해야하는 경우 `resume`과 `resumeWithException`을 사용할 수 있습니다.

#### Blocking functions

코루틴 내에서 블로킹 함수 호출 시 스레드가 차단되어 해당 스레드는 다른 코루틴 작업에 사용할 수 없게 됩니다.  
이처럼 코루틴에서 블로킹 작업이 필요한 경우 `withContext`와 `Dispatchers`를 지정해서 사용할 수 있습니다.

만약 디스패처에서 사용되는 스레드 풀의 크기를 조절하고 싶으면 `limitedParallelism`을 통해 조절할 수 있습니다.  
이를 통해 스레드의 수를 제한하거나 늘려 특정 작업에 맞는 사용자 정의 디스패처를 만들 수 있습니다. 

#### Observing with Flow

일시 중지 함수는 단일 값을 생성하거나 가져오는데 적합하지만, 여러 값을 생성하거나 가져올 땐 `Flow`가 적절합니다.

`Flow`는 다음과 같은 상황에서 사용될 수 있습니다.

- `Room`에서 테이블의 변경사항을 관찰할 때
- 실시간 메시지 수신
- 버튼 클릭이나 텍스트 변경과 같은 UI 이벤트를 관찰할 때