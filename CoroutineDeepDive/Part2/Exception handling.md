# Exception handling

코루틴 동작 방식엘서 중요한 부분 중 하나는 예외 처리입니다.

처리되지 않은 예외가 통과될 경우 프로그램이 중단되는 것처럼, 코루틴의 경우도 동일하게 중단됩니다.

예를 들어 스레드도 이러한 경우에 종료됩니다. 
그러나 스레드와 코루틴의 차이점으로는 코루틴 빌더가 부모도 취소하고, 각 취소된 부모가 모든 자식을 취소한다는 점입니다.

아래 예시를 보면 코루틴이 예외를 받으면 자신을 취소하고 예외를 부모(`launch`)에 전파합니다. 부모는 자신과 모든 자식을 취소한 다음, 그 예외를 그의 부모(`runBlocking`)에 전파합니다.

`runBlocking`은 루트 코루틴이므로 그냥 프로그램을 종료시킵니다.

```kotlin
fun main(): Unit = runBlocking {
    launch {
        launch {
            delay(1000)
            throw Error("Some error")
        }
        
        launch {
            delay(2000)
            println("Will not be printed")
        }
        
        launch {
            delay(500) // faster than the exception
            println("Will be printed")
        }
    }
    
    launch {
        delay(2000)
        println("Will not be printed")
    }
}

// Will be printed
// Exception in thread "main" java.lang.Error: Some error
```

추가적인 `launch` 코루틴을 추가해도 아무 것도 변경되지 않습니다. 
예외의 전파는 양방향입니다. 예외는 자식에서 부모로 전파되고, 그 부모가 취소되면 그들의 자식도 취소합니다.

따라서 예외 전파가 중단되지 않으면 계층 구조에서 모든 코루틴이 취소됩니다.