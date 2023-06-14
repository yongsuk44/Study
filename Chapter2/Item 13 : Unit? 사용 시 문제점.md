# 반환이나 연산 작업에서 `Unit?` 사용을 되도록 피하자

---

코틀린에서 `Unit?`은 `Unit`이나 `null` 두 가지 값만을 가질 수 있습니다.   
이는 `Boolean`이 `true`나 `false`를 가지는 것과 유사하여, 두 타입은 동치성(isomorphic)을 가집니다.  
이에 따라 `Unit?`을 `Boolean` 대신 사용해 로직을 표현하는 데 사용할 수 있습니다.

```kotlin
fun keyIsCorrect(key: String): Boolean = { ... }
if (!keyIsCorrect(key)) return
```

```kotlin
fun verifyKey(key: String): Unit? = { ... }
verifyKey(key) ?: return
```

그러나, 위와 같이 `Unit?`을 사용하면 로직을 이해하는 데 혼동을 줄 수 있으며, 오류를 감지하기 어려울 수 있습니다.  

예를 들어 아래 코드를 보죠.

```kotlin
getData()?.let { view.showData(it) } ?: view.showError()
```

`getData()`가 `null`이 아닌 값을 반환 후, `showData()`가 `null`을 반환하면, `showData()`와 `showError()`가 모두 호출됩니다. 
이는 코드를 읽는 사람에게 혼란을 줄 수 있으며, 예상치 못한 결과를 초래할 수 있습니다.

이런 혼동을 피하기 위해 표준적인 `if-else` 구문을 사용하는 것이 더 나을 수 있습니다.

```kotlin
if (person != null && person.isAdult) {
    view.showPerson(person)
} else {
    view.showError()
}
```

비교해보면 `Unit?`을 사용하는 것이 혼란스러울 수 있음을 알 수 있습니다.

```kotlin
if (!keyIsCorrect(key)) return

verifyKey(key) ?: return
```

`Unit?`을 사용하는 것이 가독성이 좋은 옵션이라는 사례는 거의 없습니다.   
대부분의 경우, `Boolean`으로 대체하는 것이 가장 좋습니다. 
그래서 일반적으로 `Unit?`를 반환하거나 작업하는 것을 피합니다.
