# Item 31: Define contract with documentation

아래 메시지를 표시하는 함수를 보면 메시지 노출 시간을 자유롭게 바꾸도록 함수를 추상화 하였습니다.

```kotlin
fun Context.showMessage(
    message: String,
    length: MessageLength = MessageLength.LONG
) {
    val toastLength = when (length) {
        SHORT -> Length.SHORT
        LONG -> Length.LONG
    }
    
    Toast.makeText(this, message, toastLength).show()
}
        
enum class MessageLength { SHORT, LONG }
```

하지만, 이 함수는 문서화 되어 있지 않습니다. 다른 개발자는 이 함수가 항상 토스트를 표시한다고 가정할 수 있습니다.
이는 구체적인 메시지 유형을 제안하지 않는 방식으로 이름을 지은 것과는 반대될 수 있습니다.
이를 명확하게 하기 위해 이 함수가 무엇을 기대하는지 설명하는 KDoc 주석을 추가하는 것이 좋습니다.

```kotlin
/**
 * 프로젝트에서 메시지를 표시하는 일반적인 방법입니다.
 * @param message 사용자에게 표시할 텍스트입니다.
 * @param length 메시지를 얼마나 오래 표시할지를 결정합니다.
 */
fun Context.showMessage(
    message: String,
    duration: MessageLength = MessageLength.LONG
) {
    val toastLength = when (length) {
        SHORT -> Length.SHORT
        LONG -> Length.LONG
    }
    
    Toast.makeText(this, message, toastLength).show()
}
        
enum class MessageLength { SHORT, LONG }
```

많은 경우, 이름만으로는 세부 사항을 명확하게 이해하기 어렵습니다.

예를 들어, `powerset`은 잘 정의된 수학적 개념이지만, 잘 알려져 있지 않으며, 해석이 충분히 명확하지 않기 때문에 설명이 필요합니다.

```kotlin
/**
 * Powerset은 원본 집합의 모든 부분집합을 포함한 집합을 반환합니다.
 * 이는 빈 집합과 원본 집합 자체를 포함합니다.
 */
fun <T> Collection<T>.powerset(): Set<Set<T>> =
    powerset(this, setOf(setOf()))

private tailrec fun <T> powerset(
    left: Collection<T>,
    acc: Set<Set<T>>
): Set<Set<T>> = when {
    left.isEmpty() -> acc
    else -> powerset(left.drop(1), acc + acc.map { it + left.first() })
}
```

이처럼 기대하는 동작을 문서화함으로 써 해결 할 수 있습니다. 이렇게 하면 개발자들은 내부 구혀넹 의존하지 않고 해당 추상화에 의존할 수 있게 됩니다.