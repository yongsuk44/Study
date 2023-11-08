# Part 1: Kotlin Coroutines 이해하기

Rxjava, Reactor와 같이 잘 정리된 기존 JVM 라이브러리들이 있습니다. 또한 Java 자체도 멀티스레딩을 지원하며, 많은 개발자들이 단순한 콜백을 사용하기도 합니다.
이처럼 명백하게 비동기 작업을 위한 여러가지 옵션들이 있습니다.

그러나 코루틴은 그 이상을 제공합니다.
코루틴은 멀티플랫폼으로 모든 Kotlin 플랫폼(JVM, JS, IOS, 공통 모듈 등)에서 사용될 수 있습니다.
또한 코드 구조를 크게 바꾸지 않습니다. 대부분의 코루틴 기능은 간단하게 사용할 수 있습니다.

## 목차

- [Why kotlin coroutines](#part-11--why-kotlin-coroutines)
- [Sequence builder](#part-12--sequence-builder)
- [How does suspendsion work](#part-13--how-does-suspendsion-work)
- [Coroutines under the hood](#part-14--coroutines-under-the-hood)
- [Coroutine - 'built in support' vs 'library'](#part-15--coroutine--built-in-support-vs-library)

## [Part 1.1 : Why Kotlin Coroutines](왜%20코루틴%20인가%3F.md)

> - Android에서 'View Update'는 'MainThread'에서만 가능, 만약 'MainThread' 차단 시 앱 크래쉬 발생
> - 이를 위해 'Thread switching', 'Callback', 'Reactive Programming' 등 제시되었으나, 각각의 문제점 존재
>   - Thread switching : 취소 메커니즘이 없고, Thread 생성은 비용이 많이 들고 Thread Switcthing의 관리가 어려움 
>   - Callback : 자연스러운 병렬 처리 미 지원, 취소 메커니즘 구현 시 추가적인 구현이 필요, 'Callback Hell' 발생 가능
>   - Reactive Programming : 코드 복잡성 증가, 명시적인 취소가 반드시 필요
> - Coroutine 도입 핵심 기능은 'Coroutine을 특정 시점에 중단 후 재개'

클라이언트 로직 중 빈번하게 처리하는 작업의 순서는 다음과 같습니다.

1. 하나 or 여러 데이터 소스(API, Database, Preference, 다른 앱 등)에서 데이터를 얻어옴
2. 데이터 처리 (파싱, 추출, 병합 등)
3. 처리된 데이터를 통해 View Update, DB Update, API 요청 등 작업 수행

여기서, 'ViewUpdate'는 Android 플랫폼에서 'MainThread'에서만 가능합니다.  
만약 'MainThread'에서 'Blocking' 작업을 하는 경우 앱이 멈출 수 있으며, 이는 앱의 크래쉬로 이어질 수 있습니다.

따라서 'Blocking' 작업을 하는 경우, 'Worker-Thread'를 통해 처리하는 것이 일반적인 관행이었습니다.  
그러나 'Thread switching'의 경우에도 문제점이 나타나게 됩니다.

1. Thread 취소 메커니즘이 없어, 빈번한 메모리 누수 현상
2. 많은 Thread 생성은 많은 비용이 발생
3. 빈번한 Thread switching은 관리가 어려움
4. 코드가 복잡해짐

위 문제를 해결하기 위한 다른 패턴으로 'Callback'이 제시되었습니다.  
'Callback' : 함수를 'non-blocking'으로 만들되, 'Callback' 프로세스가 완료되면 실행해야 하는 함수를 전달하는 것입니다.

'Callback'에서도 문제점이 나타나게 되는데 이는 다음과 같습니다.

1. 자연스러운 병렬 처리 미 지원
2. 'Callback'의 취소를 위해, 이들을 추적하고 관리하는 추가적인 코드가 필요함
3. 'Callback'의 중첩으로 인해, 'Callback Hell'이 발생할 수 있음
4. 프로그램 진행 순서를 제어하기 어려움

이를 위해 'Reactive Programming'이 제시되었으며, 이는 모든 '작업'의 시작과 처리를 'Observable DataStream' 내에서 이루어집니다.  
이런 'DataStream'은 'Thread Switching'과 'Concurrency Processing'을 지원하여 앱에서 '작업'을 병렬로 수행하는데 자주 활용됩니다. 

'Reactive Programming' 방식은 메모리 누수가 없고, 자연스러운 취소를 지원하며, 'Thread'를 효율적으로 사용하기에 'Callback' 보다 더 좋은 방법이 됩니다.  
하지만, 다음과 같은 문제점들이 생길 수 있습니다. 

1. '코드 복잡성' 커질 수 있음
2. 'DataStream'의 취소가 반드시 명시적으로 이루어져야 함
3. 'Reactive Programming' 도입 시 코드 상당 부분의 리팩토링 필요

이를 위해 'Coroutine'이 만들어지게 됩니다.

Coroutine 도입의 핵심 기능은 **Coroutine을 특정 시점에 중단 후 재개** 입니다.  
이 덕분에 'MainThread'에서 Blocking 작업 시 Thread Blocking이 아닌, 'Coroutine 중단'이 가능해집니다.  
Coroutine이 '중단'되면, 'Thread'는 'Blocking' 되지 않고, View Update 또는 다른 Coroutine을 처리하는 데 사용할 수 있습니다.

특정 작업을 호출하여 데이터를 얻는 Coroutine이 중단 후 '데이터가 준비'되면 Coroutine은 'MainThread'를 기다립니다.  
기다린 후 'MainThread'에서 Coroutine이 재개되면, '중단한 지점'부터 프로세스를 다시 진행할 수 있습니다.

## [Part 1.2 : Sequence builder](시퀀스%20빌더.md)

시퀀스 빌더는 제네레이터와 유사한 기능을 제공하며, 필요에 따라 시퀀스 항목을 제공하는 구조입니다.

요소가 요청될 떄만 계산되며, 미리 모든 요소를 생성하지 않기에 메모리 효율적입니다.  
또한 계속해서 값을 생성하는 것이 가능하여 무한한 시퀀스를 생성할 수 있습니다.

시퀀스 정의 시 `sequnece` 함수와 내부의 `yield` 함수를 사용하며, 시퀀스를 호출할 때 마다 `yield` 이후의 코드를 실행하여 다음 값을 제공합니다.  
즉, 만약 중간에서 함수가 중단되는 경우, 다음 호출 시 `yield` 이후의 코드부터 다시 실행됩니다.  
이렇게 중간에 중단되고 다시 시작하는 특성은 코루틴의 일시 정지 및 재개 메커니즘 덕분에 가능합니다.

주의할 점으로 시퀀스 빌더 내에서는 `yield` 외의 다른 일시 정지 함수를 사용할 수 없습니다.  
만약 비동기 데이터를 가져와야 하는 경우, 코루틴의 `Flow`를 사용하는 것이 더 적합합니다.

## [Part 1.3 : How does suspendsion work?](코루틴에서%20일시정지는%20어떻게%20동작될까%3F.md)

코루틴의 일시 정지와 재개는 게임의 일시 정지와 이어하기와 비슷한 맥락을 가집니다.  
**게임 일시 정지 시 현재 상태를 저장**하고 잠시 멈춘 뒤 나중에 **게임을 다시 시작할 때 다시 그 지점에서 이어서** 플레이 할 수 있습니다.

### Continuation

코루틴에서 일시 정지 함수 호출 시 코루틴이 멈추게 되며, 이 때 `Continuation`이 반환 됩니다.  
`Continuation`은 현재 코루틴의 상태를 나타내는 객체로 이를 통해 나중에 코루틴을 다시 재개할 수 있습니다.

스레드와 코루틴의 차이점으로 스레드는 차단된 경우 자원을 계속 소모하는 반면, 코루틴은 일시 정지 시 자원을 소모하지 않습니다.  
또한 코루틴의 `Continuation`은 다른 스레드에서도 재개될 수 있습니다.

### Resume with value or Exception

코루틴은 중단 후 재개될 때 새로운 데이터를 반환할 수 있으며,
이를 통해 API와 같은 비동기 작업에서 데이터를 기다리는 동안 다른 작업을 수행할 수 있어 효율적입니다.

`suspendCoroutine`을 통해 코루틴을 일시 정지할 수 있고 `Continuation`의 `resume()`을 통해 다시 재개할 수 있습니다.  
이때 `resume()`에 전달되는 값은 `suspendCoroutine`의 반환 타입과 일치해야 합니다.  
`resume`을 통해서 예외 처리 시 `resumeWithException`을 통해 예외를 전달하여 가능합니다.

### suspend 함수를 중단하는 것이 아닌, 코루틴을 중단

suspend 함수에서 일시 정지가 된 상황에서는 suspend 함수가 정지된 것이 아닌, 코루틴이 정지되었다는 점을 알고 있어야 합니다.  
이때 코루틴을 외부에서 재개하는 것은 좋은 방법이 아니며, 이러한 방식으로 재개할 경우 메모리 누수와 같은 여러 문제를 일으킬 수 있습니다.

## [Part 1.4 : Coroutines under the hood](코루틴%20내부%20동작.md)

suspend 함수는 일시 정지, 특정 위치에 다시 재개될 수 있는 함수로, 상태 기계 모델과 유사하게 동작합니다.
이는 일시 정지 호출 후 특정 상태를 지니게 되는데 이 상태를 통해 특정 지점에서 실행할 수 있습니다.
이러한 방식은 복잡한 비동기 로직을 동기식 코드처럼 보이게 만들어 유지보수와 코드 가독성에 이점을 지닙니다.

### Continuation 함수 호출 스택

코루틴은 함수의 호출과 재개를 다룰 때 연속성을 유지하기 위해 `Continuation` 객체를 사용합니다.
하나의 함수가 다른 함수를 호출 할 때, 호출된 함수의 `continuation`은 호출하는 함수의 `continuation`을 decration 하게 됩니다.
이로 인해 연결된 `continuation` 객체의 체인이 생성되며, 이 체인은 호출 스택을 나타내 활용됩니다.

### Continuation passing style

`continuation`이 인수로 함수에서 함수로 전달되는 것을 의미하며 함수 파라미터의 마지막 위치에 놓이게 됩니다.

suspend 함수를 컴파일한 코드를 보게되면 `continuation`을 마지막 파라미터로 받고 반환 값들을 `Any` | `Any?` 로 변환됨을 볼 수 있는데,
이는 suspend 함수가 `COROUTINE_SUSPENDED`이라는 코루틴이 일시 중단될 때 반환되어야 하는 특별한 값을 반환 하기에 가장 근접한 상위 타입인 `Any`를 반환하는 것입니다.

### Continuation Label & COROUTINE_SUSPENDED

코루틴은 `continuation`을 통해 일시 정지 후 재개되는데, 이때 `continuation`은 `label`을 가지게 됩니다.
`label`의 초기값은 `0`으로 함수의 시작 위치를 말하며, 그 외 숫자들은 일시 정지 후 다시 재개되어야 할 시점을 의미합니다.

코루틴이 `delay`로 일시 중단되면 `COROUTINE_SUSPENDED`를 반환하고, 이를 호출한 함수에도 `COROUTINE_SUSPENDED`를 반환하게 됩니다.
이 `COROUTINE_SUSPENDED`가 호출 스택의 최상단까지 전파되어 코루틴 빌더 또는 `resume` 함수에 도달할 때 까지 이어지며,
이러한 동작으로 현재 코루틴 작업을 하던 스레드가 코루틴이 일시정지 되었다는 것을 확인하고 다른 작업에 사용 가능하도록 처리됩니다.

## [Part 1.5 : Coroutine: Built-in support vs library](코루틴의%20구조%20지원%20vs%20라이브러리.md)

코루틴은 하나의 개념으로 참조하는 것이 일반적이지만, 내부적으로는 2가지 구성요소로 이루어져 있습니다.

### 구조 지원(Built-in Support)

Kotlin 언어 자체에서 제공하는 기능으로 매우 기본적이고 유연합니다.
그러나 이러한 유연성으로 인해 직접 사용하기 복잡하며 일반적으로 라이브러리 개발자들이 사용합니다.

### Kotlinx.coroutine

프로젝트에 별도로 추가해야하며, 구조 지원 보다 더 높은 수준의 추상화를 제공하고 있습니다.  
일반 개발자들이 사용하기 간편하고 편리하게 동시성을 다룰 수 있는 구체적인 방법을 제공합니다.