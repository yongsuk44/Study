# Dispatchers

코루틴 라이브러리에서 중요한 기능 중 하나는 코루틴이 실행되어야 할 스레드(또는 스레드 풀)을 선택할 수 있도록 하는 것입니다.

이는 디스패처(dispatchers)를 사용하여 수행되며 이를 통해 작업을 병렬로 처리하거나, 특정 작업을 메인 스레드에서 다른 스레드로 옮겨 실행 할 수 있습니다.

코루틴 컨텍스트는 코루틴의 여러 속성을 담고 있습니다. 따라서 코루틴이 어떻게 동작할지에 대한 정보들이 들어있습니다.

## Default dispatcher

디스패처를 지정하지 않으면 기본적으로 `Dispatchers.Default`가 선택됩니다.  

`Dispatchers.Default`는 다음과 같은 특징을 가지고 있습니다.
- 복잡한 수학저 계산이나 대규모 데이터의 정렬과 같은 CPU 집약적인 작업을 수행하기 위해 최적화 되어 있습니다.
- CPU 자원을 최대한 효율적으로 활용하기 위해 스레드 풀의 크기는 실행 환경의 CPU 코어 수에 따라 결정됩니다.
- 블로킹 연산을 수행할 경우, 스레드가 대기 상태에 빠져 자원을 낭비할 수 있으므로 블로킹 연산에 적합하지 않습니다.

```kotlin
suspend fun main() = coroutineScope {
    repeat(1000) {
        launch {  // or launch(Dispatchers.Default)
            List(1000) { Random.nextLong() }.maxOrNull()
            
            val threadName = Thread.currentThread().name
            println("Running on thread: $threadName")
        }
    }
}
// Running on thread: DefaultDispatcher-worker-1
// Running on thread: DefaultDispatcher-worker-5
// Running on thread: DefaultDispatcher-worker-7
// Running on thread: DefaultDispatcher-worker-6
// Running on thread: DefaultDispatcher-worker-11
// Running on thread: DefaultDispatcher-worker-2
// Running on thread: DefaultDispatcher-worker-10
// Running on thread: DefaultDispatcher-worker-4
// ...
```

`runBlocking`은 다른 디스패처가 설정되지 않은 경우 자체 디스패처를 설정합니다. 
따라서 `runBlocking` 내부에서는 자동으로 `Dispatchers.Default`가 선택되지 않습니다. 

만약 위의 예제에서 `corotineScope` 대신 `runBlocking`을 사용했다면 모든 코루틴이 메인 스레드에서 실행 됩니다.