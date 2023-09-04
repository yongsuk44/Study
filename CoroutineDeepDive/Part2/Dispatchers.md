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

---

## Limiting the default dispatcher

비용이 많이 드는 프로세스가 `Dispatchers.Default`의 모든 스레드를 사용하고 다른 코루틴들이 이 디스패처에서 스레드를 얻지 못할 것 같다고 의심이 되는 경우,
`limitedParallelism`을 사용하여 동일한 스레드에서 실행됨과 동시에 사용할 수 있는 스레드 수를 제한할 수 있습니다.

`private val dispatcher = Dispatchers.Default.limitedParallelism(5)`

이 메커니즘은 `Dispatchers.IO`를 제한하기 위한 것이지만, `Dispatchers.Default`를 제한하는 것도 가능합니다.

---

## Main dispatcher

안드로이드에서 메인 스레드는 UI 업데이트를 담당합니다. 
메인 스레드는 시간이 오래 걸리는 작업을 실행하면 앱이 멈춘 것처럼 보이므로, 긴 작업을 할 때는 다른 디스패처를 사용해야 합니다.

`Dispatchers.Main`는 UI 업데이트와 같은 작업을 메인 스레드에서 안전하게 수행할 수 있도록 도와줍니다.  
단, `Dispatchers.Main`을 사용하려면 관련된 아티팩트나 라이브러리가 프로젝트에 포함되어 있어야 하며 그렇지 않으면 사용할 수 없습니다.

프론트엔드 라이브러리는 일반적으로 유닛 테스트에서 사용되지 않으므로, 여기서는 `Dispatchers.Main`이 정의되지 않습니다.
`Dispatchers.Main`을 사용하려면 `kotlinx-coroutines-test`에서 `Dispatchers.setMain(dispatcher)`를 사용하여 디스패처를 설정해야 합니다.

```kotlin
class SomeTest {
    private val dispatcher = Executors.newSingleThreadExecutor().asCoroutineDispatcher()
    
    @Before
    fun setUp() {
        Dispatchers.setMain(dispatcher)
    }
    
    @After
    fun tearDown() {
        Dispatchers.resetMain()
        dispatcher.close()
    }
    
    @Test
    fun testSomeUI() = runBlocking {
        launch(Dispatchers.Main) {
            // ...
        }
    }
}
```

안드로이드에서는 일반적으로 `Dispatchers.Main`를 기본 디스패처로 사용합니다.

만약 블로킹 대신 suspending 기능을 가진 라이브러리를 사용하고 복잡한 계산을 하지 않는다면, 실제로는 `Dispatchers.Main`만 사용하여도 충분할 수 있습니다.
그러나 CPU 집약적인 작업을 수행한다면, 이러한 작업은 `Dispatchers.Default`에서 실행되어야 합니다.

위 2가지 디스패처만으로도 많은 앱에서는 충분하지만, 큰 파일을 읽는 것과 같이 긴 I/O 작업을 수행하는 등 스레드를 블로킹해야 하는 상황이 있을 수 있습니다.

메인 스레드를 블로킹하는 경우 앱이 멈춰버릴 수 있고 Default 디스패처를 블로킹하면 스레드 풀의 모든 스레드를 블로킹하는 위험이 있으며 어떠한 계산도 수행할 수 없게 됩니다.
이러한 상황에 대비하여 `Dispatchers.IO`가 사용됩니다.
