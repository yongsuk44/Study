## Jobs and awaiting children

[구조적 동시성](../Structured%20Concurrency.md)에서 설명한것과 같이 코루틴에서 부모 코루틴 내부에서 자식 코루틴을 생성하면 이 관계는 다음과 같은 특징을 가집니다.

- 자식 코루틴은 부모 코루틴의 컨텍스트를 상속합니다.
- 부모 코루틴은 자식 코루틴이 모두 완료될 때까지 완료되지 않습니다.
- 부모 코루틴이 취소되면 자식 코루틴도 취소됩니다.
- 자식 코루틴이 파괴되면 부모 코루틴도 파괴됩니다.

```kotlin
fun main(): Unit = runBlocking(CoroutineName("main")) {
    val name = coroutineContext[CoroutineName]?.name
    println(name) // main
    
    launch {
        delay(1000)
        val name = coroutineContext[CoroutineName]?.name
        println(name) // main
    }
}
```

이 원칙들은 대부분 `Job` 컨텍스트에 크게 의존하게 됩니다.  
`Job` 컨텍스트는 코루틴을 취소하거나, 상태를 추적하고 다른 기능을 제공할 수 있습니다.