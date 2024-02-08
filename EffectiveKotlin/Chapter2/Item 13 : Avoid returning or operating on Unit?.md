Kotlin에서 `Unit?`은 `Unit` 또는 `null` 두 가지 값만 가질 수 있다.  
이는 `Boolean`이 `true` 또는 `false` 값을 가지는 것과 비슷하다.

그래서 위 2가지 타입은 비슷하게 사용될 수 있다.   
그럼에도 `Boolean` 대신 `Unit?`을 사용하는 이유는 'Elvis 연산'과 'Safe-call'을 사용할 수 있기 떄문이다.

```kotlin
fun keyIsCorrect(key: String): Boolean = // ...

if (!keyIsCorrect(key)) return
```

위 코드를 아래와 같이 변경할 수 있다.

```kotlin
fun verifyKey(key: String): Unit? = // ...

verifyKey(key) ?: return
```

그러나, `Unit?`을 논리적 값으로 사용하는 것은 코드를 처음 보는 개발자에게 혼란을 주고, 발견하기 어려운 오류로 번질 수 있다.  

아래 예시 코드를 보자.

```kotlin
getData()?.let { view.showData(it) } ?: view.showError()
```

보통 개발자들은 `showData` 혹은 `showError` 중 하나만 실행하도록 구현 하였을 것이다.  
하지만, `getData != null && showData == null`인 경우, `showData`와 `showError`가 모두 호출된다.  
이처럼 위 코드는 가독성을 떨어뜨리고, 예상치 못한 결과를 초래할 수 있기에, 표준적인 `if-else` 구문을 사용하는 것이 더 나을 수 있다.

```kotlin
if (person != null && person.isAdult) {
    view.showPerson(person)
} else {
    view.showError()
}
```

추가로, 아래 두 표현은 기능적으로 유사하지만, 가독성과 명확성 측면에서 차이가 있다.

```kotlin
if (!keyIsCorrect(key)) return

verifyKey(key) ?: return
```

두 번째 방식은 `Unit?`을 사용하여 `verifyKey`의 성공 여부를 판단하며, 이는 첫 번째 방식보다 가독성이 떨어진다.

`Unit?`이 가독성이 좋은 옵션이라고 할 수 있는 사례가 거의 없으며, 보통은 혼란을 주기에 `Boolean`으로 대체하는 것이 좋다.
그래서 일반적으로 `Unit?`를 반환하거나 작업하는 것을 피한다.