## 리소스 관리와 `use` 함수의 활용

---
어떤 리소스들은 사용 후에 반드싣 닫혀야 하며, 닫히지 않을 경우 `memory leak`과 같은 문제가 발생 될 수 있습니다.

대표적으로 파일 입출력과 네트워크 연결등의 자원들이 이에 해당됩니다.

Java 표준 라이브러리에는 이런 리소스들을 다루기 위해 `InputStream`, `OutputStream`, `Connection`, `Reader`, `Socket`, `Scanner` 등이 있습니다.

위 클래스들은 `Closable` 인터페이스를 ㄷ구현하고 있으며, `AutoClosable`을 확장하고 있습니다.
따라서 사용이 끝난 후에는 반드시 `close()`를 호출해야 합니다.

이러한 리소스들은 대체로 무거우며, 자동으로 닫히지 않습니다. 
CG에 의해 언젠가는 처리되겠지만, 이는 시간이 소요되며, 리소스 부족 문제를 초래할 수 있습니다.

이에따라 예전부터는 `try-finally`를 사용하여 `close()`를 호출했습니다.

```kotlin
val resource = acquireResource()
try {
    // 사용할 코드
} finally {
    resource.close()
}
```

그러나 이 방식은 몇 가지 문제가 있습니다. `close()` 메소드가 예외를 던질 수 있고, 이 예외는 `catch` 블록에서 처리되지 않습니다.
또한 `try` 블록과 `finally`블록에서 모두 예외가 발생하면, 한쪽의 예외만 제대로 전파됩니다.

이런 문제점들을 해결하기 위해 Kotlin에서 `use` 함수를 제공합니다.
```kotlin
acquireResource().use { resource ->
    // 사용 코드
}
```

`use` 함수를 사용하면, 자원을 올바르게 닫고 예외를 처리하는 작업을 간결하게 처리할 수 있습니다.
또한, 파일을 1줄 씩 읽는 경우와 같이 리소스 관리가 자주 필요한 경우를 위해 Kotlin 표준 라이브러리에서는 `useLines`를 제공합니다.
`useLines` 함수는 라인별로 시퀀스를 제공하고, 처리가 완료되면 reader를 닫아줍니다.