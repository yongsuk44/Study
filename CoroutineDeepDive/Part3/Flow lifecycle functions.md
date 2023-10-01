# Flow lifecycle functions

`Flow`는 데이터 스트림을 표현하기 위한 도구로, 이 스트림은 파이프와 같은 메커니즘을 가집니다.  
이 파이프 내에서 데이터 요청과 응답은 서로 다른 방향으로 이동합니다.

`Flow` 수집 과정 중 어떤 문제나 예외가 발생하면 해당 정보는 파이프 내에 전파되며, 이를 통해 중간 동작이나 처리 단계를 종료하게 됩니다.  
예를 들어 API 호출을 통해 데이터를 가져오는 `Flow`가 있을 때, 이 API 호출에서 문제가 발생하면 해당 문제의 정보는 `Flow`를 통해 전파되고, 이후의 데이터 처리 단계는 실행되지 않습니다.

더불어 `Flow`는 다양한 이벤트를 감지하고 반응하는 기능을 제공합니다.  
Flow 수집이 시작되는 시점, 데이터가 전달되는 시점, 예외가 발생하는 시점 등 다양한 상황에 대해 로직을 삽입하거나 특정 동작을 수행할 수 있습니다.

---

## onEach

`onEach`는 일시 중지될 수 있고 요소들을 순서대로 처리합니다. 따라서 `onEach`에 `delay` 추가하면 각 값을 지연시킬 수 있습니다.

```kotlin
suspend fun main() {
    flowOf(1, 2, 3, 4)
        .onEach { print(it) }
        .collect() 
    // 1234
    
    flowOf(1,2)
        .onEach { delay(1000) }
        .collect { println(it) }
    // 1s delay 후 1 출력
    // 1s delay 후 2 출력
}
```

---

## onStart

`onStart`는 `Flow` 수집이 시작될 때 동작하는 리스너 함수로, 터미널 연산이 호출될 때 즉시 실행됩니다.  
즉, `Flow`에서 값이 흐르기 시작하기 전, 즉시 실행됩니다.

예를 들어, 특정 API로부터 데이터를 가져오는 `Flow`에서 데이터를 실제로 가져오기 시작하기 전에 로깅이나 초기화 작업을 수행하고자 할 때, `onStart`를 사용할 수 있습니다.

```kotlin
suspend fun main() {
    flowOf(1, 3)
        .onEach { delay(1000) }
        .onStart { println("Starting flow") }
        .collect { println(it) }
}
// Starting flow
// 1s delay 후 1 출력
// 1s delay 후 3 출력

suspend fun main() {
    flowOf(1, 3)
        .onEach { delay(1000) }
        .onStart { emit(0) }
        .collect { println(it) }
}
// 0
// 1s delay 후 1 출력
// 1s delay 후 3 출력
```