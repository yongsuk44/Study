'Kotlin/JVM'에서 사용하는 'Java 표준 라이브러리'에는 자동으로 닫히지 않는 리소스들이 많이 포함되어 있으며,  
이를 더 이상 사용하지 않을 떄는 수동으로 `close`를 호출해야 한다.

- `InputStream`과 `OutputStream`
- `java.sql.Connection`
- `java.io.Reader` (`FileReader`, `BufferedReader`, `CSSParser`)
- `java.new.Socket` 및 `java.util.Scanner`

위 모든 리소스들은 `Closable` 인터페이스를 구현하며, 이는 `AutoClosable`을 확장한다.

문제는 이 리소스들을 더 이상 필요하지 않을 때 반드시 `close`를 호출해야 하는데, 이 리소스들은 상대적으로 비용이 많이 들고 자체적으로 쉽게 닫히지 않는다.
CG는 참조가 유지되지 않는 경우에만 이 리소스들을 처리할 수 있지만, 이는 시간이 걸린다. 
따라서 이런 리소스들을 놓치지 않고 확실하게 닫기 위해, 전통적으로 'try-finally' 블록 안에 래핑하고 그곳에서 `close`를 호출하는 방식을 사용했다.

```kotlin
fun countCharactersInFile(path: String): Int {
    val reader = BufferedReader(FileReader(path))
    
    try {
        return reader.lineSequence().sumBy { it.length }
    } finally {
        reader.close()
    }
}
```

위와 같은 구조는 복잡하고 잘못되었는데, 그 이유는 `close`가 오류를 발생시킬 수 있으며, 해당 오류는 잡히지 않기 때문이다.  
게다가 `try`와 `finally` 양쪽에서 오류가 발생했다면, 오류는 한쪽에서만 잡히고 다른쪽에서는 무시된다.

개발자들이 기대하는 방식은 새로운 오류 정보가 이전 오류에 추가되는 것이다. 
이를 적절하게 구현하는 것은 복잡하고 길지만, 이는 흔히 발생하는 문제이므로 표준 라이브러리의 `use`로 구현 되었다.
`use`는 리소스를 적절히 닫고 예외를 처리하는데 사용되며, 모든 `Closable` 구현체에서 사용이 가능하다.

```kotlin
fun countCharactersInFile(path: String): Int {
    val reader = BufferedReader(FileReader(path))
    reader.use {
        return it.lineSequence().sumOf { it.length }
    }
}
```

또한 이 기능은 리소스 가시성을 최소화할 수 있다.

```kotlin
fun countCharactersInFile(path: String): Int {
    BufferReader(FileReader(path)).use { reader ->
        return reader.lineSequence().sumOf { it.length }
    }
}
```

파일을 한줄 씩 읽는 것과 같은 일반적인 작업도 돕기 위해, 'Kotlin 표준 라이브러리'는 `useLines`를 제공하여
파일의 각 줄을 문자열 시퀀스로 제공하고, 모든 처리가 완료되면 자동으로 기본 리더를 닫는다.

```kotlin
fun countCharactersInFile(path: String): Int {
    File(path).useLines { lines ->
        return lines.sumOf { it.length }
    }
}
```

`useLines`는 큰 파일을 처리할 때도 효율적인데, 필요에 따라 파일의 각 줄을 동적으로 읽고 메모리에는 한 줄만 유지한다.
그러나 이 시퀀스의 단점은 한 번만 사용할 수 있는데, 이는 파일의 내용을 여러 번 순회하려면 파일을 여러 번 열어야 한다는 것을 의미한다.

마지막으로 `useLines`는 표현식으로도 활용될 수 있다.

```kotlin
fun countCharactersInFile(path: String): Int =
    File(path).useLines { lines ->
        lines.sumOf { it.length }
    }
```