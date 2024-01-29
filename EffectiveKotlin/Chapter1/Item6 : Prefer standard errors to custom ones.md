`require`, `check`, `assert`는 일반적인 오류를 다룬다. 하지만 예상치 못한 다른 상황들도 표시해야 할 필요가 있다. 
예를 들어, Json 파싱 중 제공된 Json 파일이 잘못된 형식일 경우 `JsonParsingException`과 같은 사용자 정의 오류를 만드는 것이 좋다.
 
```kotlin
inline fun <reified T> String.readObject(): T {
    // ...
    if (incorrectSign) {
        throw JsonParsingException()
    }
    // ...
    return result
}
```

표준 라이브러리에서 위와 같은 상황을 나타내기에 적합한 오류가 없었기에 사용자 정의 오류를 만들었지만 
가능한 경우, 표준 라이브러리에서 제공하는 오류를 사용하는 것이 좋다.

- `IllegalArgumentException` : 함수에 전달된 'Argument'가 기대사항과 다를 때
- `IllegalStateException` : 함수의 '상태'가 기대사항과 다를 때
- `IndexOutOfBoundsException` : 컬렉션 요소에 접근하려 할 때 유효하지 않은 인덱스가 사용될 때
- `ConcurrentModificationException` : 컬렉션이나 공유 리소스를 동시에 수정할 때
- `UnsupportedOperationException` : 특정 메서드가 클래스에 존재하지만 실제로 구현되어 있지 않았거나 지원되지 않을 때
- `NoSuchElementException` : 컬렉션에서 요소를 요청했지만, 해당 요소가 존재하지 않을 때