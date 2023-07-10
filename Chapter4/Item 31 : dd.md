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

## 계약(Contract)의 중요성

계약은 사용자의 기대치와 개발자의 약속을 연결하는 매개체입니다. 사용자는 설명된 행동을 약속으로 받아들이고, 이에 따라 기대치를 조정합니다.   
이와 같은 예상 행동 전체를 '계약(Contract)'이라고 합니다.
실세계 계약과 마찬가지로, 사용자는 일단 계약이 안정화되면 개발자가 이를 이행할 것으로 기대합니다.

계약을 정의하는 것은 처음에는 두려워 보일 수 있지만, 양측 모두에게 이점이 있습니다.
잘 정의된 계약이 있다면, 개발자는 클래스가 어떻게 사용되는지 걱정하지 않아도 되고, 사용자는 내부 작동 원리에 대해 걱정할 필요가 없습니다.
사용자는 구현에 대한 지식 없이도 계약에 의존할 수 있습니다. 개발자는 계약이 준수되는 한 변경의 자유를 가집니다.

계약이 없다면, 사용자는 어떤 행동을 해야 하고, 어떤 행동을 피해야 하는지 알지 못하게 됩니다. 그 결과, 그들은 구현의 세부 사항에 의존하게 될 것입니다.
개발자는 사용자가 어떤 것에 의존하는지 모르는 상태에서, 변경을 통한 위험을 감수하거나 개발이 제한되는 문제에 직면하게 됩니다.
이러한 이유로, 계약을 명시하는 것이 중요하다는 것을 확인할 수 있습니다.

---

## 계약(Contract) 정의

계약 정의에는 다양한 방법이 존재하며 다음과 같습니다.

### 이름(name)

이름이 더 넓은 개념과 연관되어 있다면, 해당 요소가 이 개념과 일치하도록 예상합니다.
예를 들어 `sum` 메서드는 주석을 읽는것이 아닌 수학적 개념으로 접근합니다.

### 주석과 문서화(Comments and Documentation)

필요한 모든 사항을 설명할 수 있어 가장 효과적인 방법 입니다.

### 타입(Type)

타입은 객체에 많은 정보를 제공하며, 타입은 특정 메서드 집합을 명시하며, 일부 타입은 문서에서 책임을 명시하게 됩니다.
함수를 보는 경우 반환 타입과 인수 타입에 대한 정보는 매우 중요합니다.

---

## 주석 필요성 판단

Java가 처음 등장했을 때 문헌형 프로그래밍이라는 주석을 활용하여 모든 것을 상세하게 설명하는 방식이 인기를 끌었습니다.
하지만 10년이 지난 후에는 주석에 대한 강한 비판이 일어나며, 코드의 가독성 향상에 초점을 맞춰야 한다는 주장이 강해졌습니다.

하지만 주석은 함수나 클래스 같은 코드 요소를 더 높은 수준으로 설명하고, 계약을 명확히 하는 중요한 역할을 합니다.
또한, 주석은 자동 문서 생성을 위해 사용되며, 이는 프로젝트의 정보를 제공하는 중요한 부분이 될 수 있습니다.

물론, 모든 경우에 주석이 필요한 것은 아닙니다. 많은 함수들은 자체적으로 이해가 가능하며, 추가적인 설명이 필요하지 않습니다.
예를 들어, `product`는 익숙한 수학적 개념이므로, 아래 코드는 주석 없이도 충분히 이해가 가능합니다.

```kotlin
fun List<Int>.product(): Int = fold(1) { acc, i -> acc * i }
```

강조하고 싶은 내용이 이미 코드에 명확하게 드러나 있다면, 주석은 단지 방해만 됩니다.
함수의 이름과 파라미터가 이미 그 기능을 명확히 설명하고 있는 경우, 주석을 작성하지 않는 것이 좋을 수 있습니다.
아래의 예시는 함수의 이름과 파라미터 타입만으로도 기능을 유추할 수 있으므로, 추가적인 주석은 필요하지 않습니다.

```kotlin
// Product of all numbers in a list
fun List<Int>.product(): Int = fold(1) { acc, i -> acc * i }
```

또한, 코드를 구조화하는데 필요한 주석 대신 함수를 추출하는 것이 좋습니다.
아래의 `update` 함수는 더 작은 단위로 분리할 수 있는 부분이 명확합니다.

```kotlin
// before 
fun update() {
    for (user in users) {
        user.update()
    }

    for (book in books) {
        updateBook(book)
    }
}
```

```kotlin
// after
fun update() {
    updateBooks()
    updateUsers()
}

private fun updateBooks() {
    for (book in books) {
        updateBook(book)
    }
}

private fun updateUsers() {
    for (user in users) {
        user.update()
    }

}
```

그러나, 주석이 중요하고 유용한 경우도 많이 있습니다.
예를 들어, Kotlin 표준 라이브러리의 대부분의 공개 함수들은 자유롭게 활용 가능하도록 잘 정의된 계약을 가지고 있습니다.

```kotlin
/**
 * Returns a new read-only list of given elements.
 * The returned list is serializable (JVM).
 */
public fun <T> listOf(vararg elements: T): List<T> =
    if (elements.size > 0) elements.asList() else emptyList()
```

`listOf` 함수를 보면, 이 함수는 읽기 전용이며 JVM에서 직렬화가 가능한 `List`를 반환한다는 것을 명시하고 있습니다.

---

## KDoc 형식

함수를 주석으로 문서화 할 때 공식적으로 사용하는 형식을 Kdoc이라 합니다.  
Kdoc에서는 마크다운 형식으로 설명을 작성하도록 합니다.

Kdoc의 구조는 다음과 같습니다.

- 문서의 첫 번째 문단은 요소의 요약 설명
- 문서의 두 번째 부분은 요소의 상세 설명
- 그 후의 각 줄은 태그로 시작하며, 태그들은 요소를 설명하기 위해 사용

Kdoc에서 지원하는 태그들은 다음과 같습니다.

| 태그                  | 설명                                                                                               |
|---------------------|--------------------------------------------------------------------------------------------------|
| @param              | 함수의 값 파라미터, 타입 파라미터, 클래스, 속성을 문서화합니다.                                                            |
| @return             | 함수의 반환 값을 문서화합니다.                                                                                |
| @constructor        | 클래스의 기본 생성자를 문서화합니다.                                                                             |
| @receiver           | 확장 함수의 리시버를 문서화합니다.                                                                              |
| @property           | 특정 이름을 가진 클래스의 속성을 문서화합니다. 이는 주로 주 생성자에서 정의된 속성에 사용됩니다.                                          |
| @throws, @exception | 메서드에서 발생 가능한 예외를 문서화합니다.                                                                         |
| @sample             | 특정한 정규 이름을 가진 함수의 본문을 현재 요소의 문서화에 포함시킵니다. 이는 요소를 어떻게 사용할 수 있는지 예제를 보여줍니다.                        |
| @see                | 특정한 클래스나 메서드에 대한 링크를 추가합니다.                                                                      |
| @author             | 문서화된 요소의 저자를 표시합니다.                                                                              |
| @since              | 문서화된 요소가 도입된 버전을 표시합니다.                                                                          |
| @suppress           | 생성된 문서에서 특정 요소를 제외시킵니다. 이는 주로 공식 API에 포함되지 않은 모듈의 요소에 대해 사용되며, 이 요소들은 외부에 노출되어야 하지만 문서화에서 제외됩니다. |

이러한 모든 태그들은 Kotlin 문서 생성 도구에 의해 이해될 수 있습니다. 공식 도구는 Dokka라고 합니다. 
Dokka는 온라인으로 게시하고 외부 사용자에게 제공될 수 있는 문서화 파일을 생성합니다.
