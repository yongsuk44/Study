# Item 12: 연산자의 의미는 그 함수 이름과 일치해야 한다

---

연산자 오버로딩은 강력한 기능이지만, 대부분의 강력한 기능들처럼 이 또한 위험할 수 있습니다.

예를 들어, 숫자의 팩토리얼을 계산하는 함수를 만든다고 가정을 해보죠

```kotlin
fun Int.factorial(): Int = (1..this).product()
fun Iterable<Int>.product(): Int = fold(1) { acc, i -> acc * i }
```

이 함수는 `Int`에 대한 확장 함수로 정의되어 있으므로 사용하기 편리합니다.

```kotlin
print(10 * 6.factorial()) //7200
```

### 잘못된 연산자 오버로딩 예

팩토리얼의 수학적 표현은 숫자 뒤에 느낌표(!)를 붙이는 것입니다.

```kotlin
10 * 6!
```

Kotlin에는 이런 연산자가 없지만, not 연산자를 오버로딩하여 팩토리얼을 표현하는 것이 가능합니다.

```kotlin
operator fun Int.not(): Int = factorial()

print(10 * !6)//7200
```

그러나 이렇게 해야 하는가에 대한 가장 단순한 답변은 '아니요'입니다. 함수 선언만 보아도 이 함수의 이름이 `not`임을 알 수 있습니다.
이 함수는 논리 연산을 나타내는 것이며, 이를 수학적 팩토리얼 표현에 사용하는 것은 혼동을 가져올 수 있습니다.

### Kotlin에서의 연산자 의미

```kotlin
a - b // a.minus(b)
a / b // a.div(b)
a += b // a.plusAssign(b)
a > b // a.compareTo(b) > 0
```

Kotlin에서 각 연산자의 의미는 항상 동일합니다.   
이는 중요한 설계 결정이며, 이로 인해 어떤 라이브러리를 처음 사용할 때도 함수와 클래스의 의미를 쉽게 이해할 수 있습니다.   
연산자가 다른 의미를 가지는 경우, 각 연산자를 별도로 이해하고, 해당 상황에서 무슨 의미인지 기억해야 하므로 복잡해질 수 있습니다.   
하지만 Kotlin에서는 각 연산자가 구체적인 의미를 가지고 있으므로 이런 문제가 발생하지 않습니다.

---

## 연산자 의미와 함수 이름이 명백하지 않은 경우

가장 큰 문제는 특정 사용법이 Convention을 준수하는지 여부가 불분명한 경우입니다.

예를 들어, 함수를 세 배로 만들었을 때 이것이 무엇을 의미하는지 불분명할 수 있습니다.

```kotlin
operator fun Int.times(operation: () -> Unit): () -> Unit = {
    repeat(this) { operation() }
}
val tripledHello = 3 * { print("Hello") }
tripledHello() // HelloHelloHello
```

어떤 사람들에게는, 위 코드는 이 함수를 3번 반복하는 새로운 함수를 만드는 것을 의미하는 것입니다.  
하지만 또 다른 사람들에게는, 아래 코드는 이 함수를 3번 호출하고 싶다는 것을 의미하는 것일 수 있습니다.

```kotlin
operator fun Int.times(operation: () -> Unit): () -> Unit = {
    repeat(this) { operation() }
}

3 * { print("Hello") } // HelloHelloHello
```

의미가 불분명한 경우, 기술적인 확장 함수를 선호하는 것이 좋습니다.
또한 사용법을 연산자처럼 유지하려면, infix를 사용할 수 있습니다.

```kotlin
operator fun Int.times(operation: () -> Unit): () -> Unit = {
    repeat(this) { operation() }
}
val tripledHello = 3 timesRepeated { print("Hello") }
tripledHello() // HelloHelloHello
```

때때로 상위 수준의 함수를 사용하는 것이 더 좋습니다.   
함수를 3번 반복하는 것은 이미 구현되어 있고 표준 라이브러리에서 제공됩니다.

```kotlin
repeat(3) { println("Hello") } // HelloHelloHello
```

---

## "연산자 의미 = 함수 이름"에 대한 룰은 언제 깨도 괜찮은지

이러한 경우 연산자 오버로딩을 이상한 방식으로 사용해도 괜찮습니다.   
`Domain Specific Language`(DSL)을 설계할 때입니다.

클래식한 HTML DSL 예제를 생각해보죠

```kotlin
body {
    div {
        +"Some Text"
    }
}
```

여기서, 요소에 텍스트를 추가하려면 `String.unaryPlus`를 사용하는 것을 볼 수 있습니다.   
이것은 명확하게 `Domain Specific Language`의 일부이기 때문에 허용됩니다.

---

## 정리

연산자 오버로딩은 신중하게 사용하세요. 함수 이름은 항상 그 행동과 일관되어야 합니다.   
연산자의 의미가 불분명한 경우는 피하고, 대신에 기술적인 이름을 가진 일반 함수를 사용하여 명확하게 만드세요.   
더 연산자처럼 보이는 구문을 원한다면, infix 수식어나 상위 수준의 함수를 사용하세요.