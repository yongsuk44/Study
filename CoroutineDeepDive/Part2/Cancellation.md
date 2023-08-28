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