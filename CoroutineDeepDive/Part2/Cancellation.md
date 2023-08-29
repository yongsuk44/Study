# Cancellation

코루틴의 중요한 기능 중 하나는 취소(Cancellation) 입니다.   
단순히 스레드를 죽이는 것은 리소스를 해제할 기회를 제공하지 않으므로 좋은 해결책이 아닙니다.

코루틴은 편리하고 안전한 취소 메커니즘을 제공하여 이러한 문제를 해결합니다.

## Basic cancellation

`Job` 인터페이스에는 취소 기능을 위한 `cancel` 메서드를 제공하며 이를 호출하면 다음과 같은 효과가 발생합니다.

#### 1. 첫 번째 suspension point에서 작업 종료

코루틴이 실행 중일 때 `cancel`메서드 호출 시, 코루틴은 다음 suspension point에서 작업을 종료합니다.  
예를 들어 `delay` or `yield` 같은 함수를 호출하여 코루틴이 일시 정지되는 지점이 있는 경우 그 지점에서 작업이 종료됩니다.

#### 2. 하위 작업 취소

`Job` 인터페이스를 구현하는 객체는 자식 작업을 가질 수 있습니다.   
이처럼 구조적 동시성 메커니즘을 위해서 부모 작업을 취소하면 자식 작업도 자동으로 취소됩니다.

예를 들어 A-B-C 작업 중 B가 취소되면 B,C가 취소되고 A는 아무런 영향을 받지 않습니다.

#### 3. 취소된 작업의 상태 변화

작업이 취소되면, 2가지 상태 변화를 겪습니다. 처음에는 `CANCELLING` 상태가 되고 그 다음 `CANCELLED` 상태가 됩니다.  
또한 취소된 작업은 더 이상 새로운 코루틴의 부모로서 사용될 수 없습니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = launch {
        repeat(1_000) { i ->
            delay(200)
            println("Printing $i")
        }
    }

    delay(1100)
    job.cancel()
    job.join()
    println("Cancelled Success")
}

// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled Success
```

`cancel` 메서드는 예외 객체를 인자로 받을 수 있으며 이를 통해 코루틴이 왜 취소되었는지 명확하게 알 수 있습니다.

그러나 이 예외는 반드시 `CancellationException`의 하위 타입이어야 합니다.   
이는 코루틴을 안전하게 취소하기 위한 제약 사항 입니다.

만약 `CancellationException` 외 다른 타입의 예외로 코루틴을 취소하려고 시도하면 코루틴은 취소되지 않습니다. 

`join` 메서드는 코루틴의 작업이 완전히 종료될 때까지 기다립니다.  
코루틴을 취소한 후 `join` 호출 시 취소가 완전히 처리되고 나서야 다음 작업을 수행할 수 있습니다.

이렇게 하지 않으면 race condition 즉, 코루틴이 완전히 취소되기 전에 다른 코드가 실행될 가능성이 있습니다. 
이로 인해 예기치 않은 결과 혹은 버그가 발생될 수 있습니다.

아래 예제는 `join`이 없으면 `Cancelled success` 출력 후 `Printing 4`가 출력되는 것을 볼 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    val job = luanch {
        repeat(1_000) { i ->
            delay(200)
            Thread.sleep(100) // We simulate long operation
            println("Printing $i")
        }
    }

    delay(1000)
    job.cancel()
    println("Cancelled success")
}

// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Cancelled Success
// Printing 4
```

위 코드에서 `join` 추가 시 코루틴이 취소가 완료될 때까지 일시 정지되어 원하던 결과를 얻을 수 있습니다.

```kotlin
suspend fun main() = coroutineScope {
    val job = luanch {
        repeat(1_000) { i ->
            delay(200)
            Thread.sleep(100) // We simulate long operation
            println("Printing $i")
        }
    }

    delay(1000)
    job.cancel()
    job.join()
    println("Cancelled success")
}

// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled Success
```

`kotlinx.coroutines`에서는 `cancel`과 `join`을 같이 쉽게 호출하도록 `cancelAndJoin`이라는 편리한 확장 함수를 제공합니다.

```kotlin
// The most explicit function name I've ever seen
public suspend fun Job.cancelAndJoin() {
    cancel()
    return join()
}
```

`Job()` 팩토리 함수를 사용하여 생성된 작업도 동일한 방식으로 취소할 수 있으며 이는 종종 여러 코루틴을 한번에 쉽게 취소하는 데 사용됩니다.

```kotlin
suspend fun main() = coroutineScope {
    val job = luanch {
        repeat(1_000) { i ->
            delay(200)
            Thread.sleep(100) // We simulate long operation
            println("Printing $i")
        }
    }

    delay(1100)
    job.cancelAndJoin()
    println("Cancelled success")
}

// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled Success
```

이러한 방식은 중요합니다. 

많은 플랫폼에서는 사용자가 View를 떠날 때 View에서 시작된 모든 코루틴을 취소하는 등 동시에 실행되는 여러 작업을 취소해야하는 경우가 자주 있습니다.

```kotlin
class ProfileViewModel : ViewModel() {
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())

    fun onCreate() {
        scope.launch { loadUserData() }
    }

    override fun onCleared() {
        scope.coroutineContext.cancelChildren()
    }
}
```

---

## How does cancellation work?

`Job`이 취소되면 상태가 `CANCELLING`으로 변경됩니다. 이 상태에서 코루틴이 다음 suspension point에 도달하면 `CancellationException`이 발생합니다.

`CancellationException`은 `try-catch`를 사용하여 잡을 수 있습니다.
그러나 이 예외를 잡아서 특별한 처리를 하지 않는 한, 보통은 잡은 예외를 다시 던져야 코루틴의 취소 상태가 정상적으로 전파됩니다.

`CancellationException`을 다시 던지는 것은 코루틴의 부모나 상위 작업이 이 코루틴이 취소되었다는 것을 전파하기 위해서 입니다.
만약 이렇게 처리하지 않으면 코루틴의 취소가 제대로 처리되지 않을 수 있습니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    
    launch(job) {
        try {
            repeat(1_000) { i -> 
                delay(200)
                println("Printing $i")
            }
        } catch (e: CancellationException) {
            println("Cancelled")
            throw e
        }
    }
    
    delay(1100)
    job.cancelAndJoin()
    println("Cancelled Success")
    delay(1000)
}

// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// JobCancellationException
// Cancelled Success
```

취소된 코루틴은 내부적으로 `CancellationException`이 발생하는 것을 이해할 수 있습니다.  
이는 코루틴이 강제로 종료되는 것이 아니라 정상적인 예외 처리 흐름을 따르게 하여 리소스를 안전하게 정리할 수 있는 기회를 줍니다.

코루틴이 취소되거나 예외가 발생되더라도 `finally` 블록은 항상 실행됩니다.   
이 블록에서 파일을 닫거나, 데이터베이스 연결을 종료하는 등의 리소스 해제나 필요한 정리 작업을 수행할 수 있습니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    
    launch(job) {
        try {
            delay(Random.nextLong(2000))
            println("Done")
        } finally {
            print("Will always be printed")
        }
    }
    delay(1000)
    job.cancelAndJoin()
}
// Will always be printed
// Done 
// Will always be printed
```

---

## Just one more call

`CancellationException`을 확인하고 코루틴이 완전히 종료되기 전 추가 작업을 수행할 수 있다면 어디까지 할 수 있는지 궁금할 수 있습니다.

이때 코루틴은 모든 리소스를 정리할 수 있을 만큼 오래 실행할 수 있습니다. 그러나, 일시 정지는 더 이상 허용되지 않습니다.  
왜냐하면 `Job`은 이미 `CANCELLING` 상태에 있으며, 이 상태에서는 새로운 코루틴을 시작하려고 하면 무시될 것이고 일시 정지하려고 하면 `CancellationException`이 발생됩니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    
    launch(job) {
        try {
            delay(2000)
            println("Job is done")
        } finally {
            println("Finally")
            
            launch { // will be ignored 
                println("Will not be printed")
            }
            
            delay(1000) // here exception is thrown 
            println("Will not be printed")
        }
    }
    
    delay(1000)
    job.cancelAndJoin()
    println("Cancel done")
}
// 1s delay
// Finally
// Cancel done
```

때때로 코루틴이 이미 취소된 상태에서 일시 정지 함수를 사용해야 할 필요가 있습니다.
예를 들어 데이터베이스의 변경 사항을 롤백하는 등의 상황에서는 이러한 호출이 필요할 수 있습니다.

이런 경우에는 `withContext(NonCancellable)` 함수로 이러한 호출을 감싸는 것이 좋습니다.

`withContext` 함수는 코드 블록의 컨텍스트를 변경합니다.   
`withContext`내에서 `NonCancellable` 객체를 사용했는데 이 객체는 취소할 수 없는 `Job`입니다.
따라서 블록 내에서 작업은 `ACTIVE` 상태에 있으며, 원하는 일시 정지 함수를 호출할 수 있습니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    
    launch(job) {
        try {
            delay(2000)
            println("coroutine finished")
        } finally {
            println("Finally")
            
            withContext(NonCancellable) {
                delay(1000)
                println("Cleanup done")
            }
        }
    }
    
    delay(100)
    job.cancelAndJoin()
    println("done")
}
// Finally
// Cleanup done
// done
```

---

## invokeOnCompletion

`invokeOnCompletion` 함수는 코루틴 `Job`이 `COMPLETED` 또는 `CANCELLED`와 같이 최종 상태에 도달했을 때 실행할 코드 블록(Handler)을 설정하는데 사용됩니다.
이는 리소스를 해제하거나, 다른 코루틴에 알림을 보내는 등의 작업에 유용합니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = launch {
        delay(1000)
    }
    
    job.invokeOnCompletion { exception: Throwable? -> 
        println("Finished")
    }
    delay(400)
    job.cancelAndJoin()
}
// Finished
```

위 예제처럼 `invokeOnCompletion`의 코드 블록(Handler)의 예외를 파라미터로 받을 때 코루틴이 어떻게 종료되었는지 판단할 수 있습니다.

- `null` : 코루틴이 정상적으로 종료
- `CancellationException` : 코루틴이 취소됨
- 그 외 예외 : 코루틴이 예외와 함께 종료됨

만약 `invokeOnCompletion` 호출 전에 코루틴이 이미 완료된 경우, 설정한 코드 블록(handler)이 즉시 실행됩니다.
이는 예기치 않은 상황에서도 코드 블록이 실행되도록 보장됩니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job= launch {
        delay(Random.nextLong(2400))
        println("Finished")
    }
    delay(800)
    job.invokeOnCompletion { exception: Throwable? ->
        println("Will always be printed")
        println("The exception was: $exception")
    }
    delay(800)
    job.cancelAndJoin()
}
// Will always be printed
// The exception was: kotlinx.coroutines.JobCancellationException

// or

// Finished
// Will always be printed
// The exception was: null
```

`invokeOnCompletion`은 코루틴이 취소되는 순간 동기적으로 호출되며, 이는 다른 스레드에서도 실행될 수 있습니다.
즉, `invokeOnCompletion`이 실행되는 스레드를 직접 제어할 수 없습니다.

---

## Stopping the unstoppable

취소는 일반적으로 코루틴의 suspension point에서 발생하며 이러한 point가 없으면 취소가 발생되지 않습니다.

예를 들어 `delay` 대신 `Thread.sleep`을 사용하면 위와 같은 상황을 테스트할 수 있습니다.

실제로 복잡한 계산이나 블로킹 I/O 작업을 하는 경우 코루틴을 적절히 중단하지 않으면 취소가 어려울 수 있습니다.

아래 예시는 코루틴 내부에 suspension point가 없어 취소할 수 없는 상황으로 실행 시간은 3분 이상이지만, 실제로는 1초 후 취소되어야 합니다.

    이러한 상황은 테스트를 위한 환경이니 실제 프로젝트에서는 사용하면 안됩니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        repeat(1_000) { i ->
            Thread.sleep(200) // We might have some
            // complex operations or reading files here
            println("Printing $i")
        }
    }
    delay(1000)
    job.cancelAndJoin()
    println("Cancelled sucess")
    delay(1000)
}
// Printing 0
// Printing 1
// Printing 2
// ... (up to 1000)
```

위와 같은 상황에서 대처할 수 있는 방법은 여러가지가 있습니다.

첫 번째 방법으로 `yield()`를 가끔 사용하는 것입니다.

`yield()`는 코루틴을 잠깐 중단하고 다시 재개합니다. 이렇게 하면 코루틴이 취소 명령을 수신할 수 있는 suspension point가 생성됩니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        repeat(1_000) { i ->
            Thread.sleep(200)
            yield()
            println("Printing $i")
        }
    }
    delay(1100)
    job.cancelAndJoin()
    println("Cancelled sucess")
    delay(1000)
}
// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled sucess
```

`yield()`를 사용하는 것은 좋은 관행으로 특히, 중단되지 않은 CPU-Intensive 또는 Time-Intensive 작업의 블록 사이에 사용하는 것이 좋습니다.

이렇게하면 코루틴이 더 효율적으로 동작하고, 필요한 경우 코루틴을 더 쉽게 취소할 수 있습니다.

```kotlin
suspend fun cpuIntensiveOperation() = withContext(Dispatchers.Default) {
    cpuIntensiveOperation1()
    yield()
    cpuIntensiveOperation2()
    yield()
    cpuIntensiveOperation3()
}
```

또 다른 방법으로는 `Job`의 상태를 추적하는 것입니다.

코루틴의 `Job` 상태를 모니터링하려면 `isActive` 속성을 사용하여 현재 코루틴이 `ACTIVE` 상태인지 아닌지를 빠르게 판단할 수 있게 도와줍니다.

```kotlin
public val CoroutineScope.isActive: Boolean 
    get() = coroutineContext[Job]?.isActive ?: true
```

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        do {
            Thread.sleep(200)
            println("Printing")
        } while(isActive)
    }
    delay(1100)
    job.cancelAndJoin()
    println("Cancelled sucess")
}

// Printing
// Printing
// Printing
// Printing
// Printing
// Cancelled sucess
```

또는 `ensureActive()`를 사용하여 `Job`이 `ACTIVE` 상태가 아닌 경우 `CancellationException`을 발생시킬 수 있습니다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        repeat(1000) { num ->
            Thread.sleep(200)
            ensureActive()
            println("Printing $num")
        }
    }
    delay(1100)
    job.cancelAndJoin()
    println("Cancelled successfully")
}
// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled successfully
```

`ensureActive()`와 `yield()`의 결과는 유사해 보이지만 실제로는 매우 다릅니다.

`ensureActive()`는 `CoroutineScope`(또는 `CoroutineContext`, `Job`)에서 호출되어야 합니다.
`Job`이 더 이상 `ACTIVE` 상태가 아닌 경우 예외를 발생시키는 것 외에는 아무런 행동도 하지 않습니다.

`yield()`는 일반적인 최상위 suspension 함수입니다. 따라서 어떤 범위도 필요하지 않으므로 일반 suspension 함수에서 사용할 수 있습니다.
또한 중단과 재개를 수행하므로 스레드 풀이 있는 디스패처를 사용하는 경우 스레드 변경이 발생되는것과 같이 다른 효과가 발생할 수 있습니다.

---

## suspendCancellableCoroutine

`suspendCancellableCoroutine` 함수는 `suspendCoroutine`과 비슷하게 동작하지만, 
`continuation`은 `CancellableContinuation<T>`로 래핑되어 몇 가지 추가적인 메서드를 제공합니다.

가장 중요한 메서드는 `invokeOnCancellation`이며, 이를 통해 코루틴이 취소될 때 어떤 작업을 수행해야 하는지를 정의할 수 있습니다.

```kotlin
suspend fun someTask() = suspendCancellableCoroutine { cont -> 
    cont.invokeOnCancellation { 
        // do cleanup
    }
    
    // rest of the implementation
}
```

다음은 Retrofit Call을 suspension 함수로 감싸는 예제 입니다.

```kotlin
suspend fun getOrganizationRepos(
    organization: String
): List<Repo> = suspendCancellableCoroutine { continuation -> 
        val orgReposCall = apiService.getOrganizationRepos(organization)
        
        orgReposCall.enqueue(object: Callback<List<Repo>> {
            override fun onResponse(call: Call<List<Repo>>, response: Response<List<Repo>>) {
                if (response.isSuccessful) {
                    val body = response.body()
                    
                    if (body != null) continuation.resume(body)
                    else continuation.resumeWithException(ResponseWithEmptyBody)
                } else {
                    continuation.resumeWithException(
                        ApiException(
                            response.code(), 
                            response.message()
                        )
                    )
                }
            }
            
            override fun onFailure(call: Call<List<Repo>>, t: Throwable) {
                continuation.resumeWithException(t)
            }
        })
        
        continuation.invokeOnCacnellation { orgReposCall.cancel() }
    }

class GithubApi {
    @GET("orgs/{organization}/repo?per_page=100")
    suspend fun getOrganizationRepos(
        @Path("organization") organization: String
    ): List<Repo>
}
```

`CancellableContinuation<T>`은 `Job`의 상태를 확인(`isActive`, `isCompleted`, `isCancelled` 프로퍼티 사용)하고 취소 원인과 함께 `continuation`을 취소할 수 있게 해줍니다.