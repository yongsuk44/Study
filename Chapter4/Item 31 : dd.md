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

---

## 계약의 중요성

계약은 사용자의 기대치와 개발자의 약속을 연결하는 매개체입니다. 사용자는 설명된 행동을 약속으로 받아들이고, 이에 따라 기대치를 조정합니다.   
이와 같은 예상 행동 전체를 '계약(Contract)'이라고 합니다. 
실세계 계약과 마찬가지로, 사용자는 일단 계약이 안정화되면 개발자가 이를 이행할 것으로 기대합니다.

계약을 정의하는 것은 처음에는 두려워 보일 수 있지만, 양측 모두에게 이점이 있습니다. 
잘 정의된 계약이 있다면, 개발자는 클래스가 어떻게 사용되는지 걱정하지 않아도 되고, 사용자는 내부 작동 원리에 대해 걱정할 필요가 없습니다. 
사용자는 구현에 대한 지식 없이도 계약에 의존할 수 있습니다. 개발자는 계약이 준수되는 한 변경의 자유를 가집니다.

계약이 없다면, 사용자는 어떤 행동을 해야 하고, 어떤 행동을 피해야 하는지 알지 못하게 됩니다. 그 결과, 그들은 구현의 세부 사항에 의존하게 될 것입니다. 
개발자는 사용자가 어떤 것에 의존하는지 모르는 상태에서, 변경을 통한 위험을 감수하거나 개발이 제한되는 문제에 직면하게 됩니다. 
이러한 이유로, 계약을 명시하는 것이 중요하다는 것을 확인할 수 있습니다.