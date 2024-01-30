아래와 같은 경우에, 때때로 함수에서 원하는 결과를 생성할 수 없는 경우가 있다.

- 서버에서 데이터를 얻을때, 인터넷 연결에 문제가 생긴 경우
- 조건을 충족하는 첫 번째 요소를 얻을 때, 목록에 해당 요소가 없는 경우
- 텍스트에서 객체를 파싱할 때, 텍스트 형식이 잘못된 경우

이런 상황을 처리하는 두 가지 메커니즘이 있다.

- `null` or `sealed class`(`Failure class`) 반환
- Throw an exception

위 두 가지 메커니즘 사이에는 중요한 차이점이 있다.

예외는 정보를 전달하는 표준 방법으로 사용되어서는 안되며, 부정확하고 특별한 상황을 나타내기 위해 사용되어야 한다.
즉, 예외는 예외 조건이 부합할 때 사용해야 하며, 그 이유는 아래와 같다.

- 예외가 전파되는 방식은 가독성이 낮고 이해하기 어렵다. 
  - 복잡한 시스템의 경우 예외가 어디서 발생하고 어디로 전파되는지 추적하기 어렵기에 예외 처리가 누락되거나 잘못 처리될 수 있다.
- Kotlin에서는 모든 예외는 기본적으로 'unchecked' 이다. 
  - 컴파일러가 예외 처리를 강제하지 않아, 예외 조건을 알 수 없는 경우가 많기에, 예외 조건이 명확할 때 사용되어야 한다.
- 예외는 예외적인 상황을 위해 설계 되어 있다.
  - JVM 환경에서 예외 처리 시 '여러 비용'들이 발생하기에 명시적인 테스트들(`if`, `when` 등 조건문)만큼 효율적이지 않아, 예외 조건에 부합될 때 사용되어야 한다.
  - 여러 비용 : 스택 트레이스 처리 비용, 예외 객체 생성 비용, 예외 처리 비용 등
- `try-catch` 블록 안에 코드를 배치하면, 컴파일러가 수행 할 수 있는 특정 최적화 작업을 방해한다.
  - 최적화 작업 : Inline 함수 호출, 불필요 코드 제거 등 

반면, `null` or `Failure`는 예상 가능한 오류를 나타내기에 적합하다. 이들은 명시적이고, 효율적이며, 관용적인 방식으로 처리될 수 있다.

위와 같은 이유로 인해 아래 규칙을 지켜야 한다.

- 오류가 예상될 때, `null` or `Failure` 반환
- 오류가 예상 되지 않을 때, Throw exception

```kotlin
sealed class Result<out T> {
    data class Success<out T>(val result: T) : Result<T>()
    data class Failure(val throwable: Throwable) : Result<Nothing>()
}

class JsonParsingException : Exception()

inline fun <reified T> String.readObjectOrNull(): T? {
    // ...
    if (incorrectSign) {
        return null
    }
    // ...
    return result
}

inline fun <reified T> String.readObject(): Result<T> {
    // ...
    if (incorrectSign) {
        return Failure(JsonParsingException())
    }
    // ...
    return Success(result)
}

```

위와 같은 방식으로 나타나는 오류는 처리하기 쉽고 놓치기 어렵다.
'null'을 사용하기로 선택할 때, 이런 값을 처리하는 클라이언트는 'Null-safety' 기능 중 'safe-call' or 'Elvis 연산'을 사용하여 'null'을 처리 할 수 있다.

```kotlin
val age = userText.readObjectOrNull<Person>()?.age ?: -1
``` 

`Result`와 같은 'Union type'을 사용하기로 선택할 때, 아래와 같이 `when`을 사용하여 처리 할 수 있다.

```kotlin
val age =
    when (val person = userText.readObject<Person>()) {
        is Success -> person.age
        is Failure -> -1
    }
```

위와 같은 오류 처리 방식은 `try-catch` 보다 더 효율적일 뿐만 아니라, 사용하기도 더 쉽고 명시적이다.
예외가 발생하고, 이를 적절하게 처리하지 않으면 앱 전체가 중단될 위험이 있다.
반면, `null` or `sealed class`는 명시적으로 처리해야 하지만 앱을 중단 시키지 않는다.

`Failure`는 어떤 데이터도 포함시킬 수 있기에 실패한 상황에서 정보를 추가해서 전달할 필요가 있는 경우,
`sealed class`를 선호하는 것이 좋고, 그렇지 않은 경우 `null`을 사용하는 것이 좋다.

함수가 실패를 다룰 때 두 가지 버전을 가지는 것이 일반적이다. 이에 대한 좋은 예시로 `List`가 있다.

- `get`은 주어진 인덱스에 요소가 있을 것이라 예상되고 사용되며, 요소가 없는 경우 `IndexOutOfBoundsException`을 발생시킨다.
- `getOrNull`은 '목록의 range' 밖 요소를 요청할 수 있다고 가정하고 사용되며, 요소가 없는 경우 `null`을 반환한다.

또한 `getOrDefault`와 같은 다른 옵션도 지원하지만, 일반적으로 `getOrNull`과 'Elvis 연산'을 통해 쉽게 대체될 수 있다. 