## 동시성 구조화(Structured Concurrency)

구조화된 동시성은 코드의 실행 흐름을 조직화하고, 동시성 코드의 유지보수를 용이하게 만드는 프로그래밍 패러다임 중 하나입니다.  
이는 코루틴이 서로에게 어떻게 종속되는지를 명확하게 하며, "리소스의 유출"이나 "비정상 종료" 같은 문제를 방지합니다.

- 부모 코루틴 : 새로운 코루틴을 시작하는 코루틴
- 자식 코루틴 : 부모 코루틴 내에서 시작된 코루틴

부모 코루틴이 종료될 때 모든 자식 코루틴도 자동으로 취소되어 종료됩니다.  
이로 인해 관련된 코루틴들의 생명주기를 간편하게 관리할 수 있습니다.

### CoroutineScope

`CoroutineScope`는 코루틴의 생명주기를 제어하고 관리하는 `Context`를 제공합니다. 
스코프 내에서 시작된 코루틴들은 서로 부모-자식 관계를 형성합니다.

```kotlin
fun main() = runBlocking { // 부모 코루틴
    launch {
        // 자식 코루틴 
    }
}
```