대부분의 강한 기능은 위험성을 함께 동반하고 있으며, 'Operator overloading'도 그 중 하나이다.  
프로그래밍에서 큰 힘은 동시에 큰 책임이 따르며, 'Operator overloading'를 처음 사용할 때 그 기능에 쉽게 빠져들 수 있다.

예를 들어, 팩토리얼을 계산하는 함수를 만들어보면 다음과 같을 수 있다.

```kotlin
fun Int.factorial(): Int = (1..this).product()

fun Iterable<Int>.product(): Int = fold(1) { acc, i -> acc * i }
```

위 함수들은 `Int`에 대한 확장 함수로 정의되어 있어 사용하기 편리하다.

```kotlin
print(10 * 6.factorial()) //7200
```

팩토리얼의 수학적 표기법은 숫자 뒤에 느낌표(!)를 붙여서 표현한다.

```kotlin
10 * 6!
```

Kotlin에는 이런 연산자를 지원하지 않지만, not 연산자를 오버로딩하여 팩토리얼을 표현하는 것이 가능하다.

```kotlin
operator fun Int.not(): Int = factorial()

print(10 * !6) //7200
```

위와 같은 작업을 할 수는 있지만, 실제로는 사용해서는 안된다. 함수의 선언만 보아도, 팩토리얼에 대한 기능을 수행하기애 적절하지 않음을 알 수 있다.
또한 함수 이름에서 알 수 있듯이, 수치 계산이 아닌 논리 연산을 대표하기에, 이를 수학적 팩토리얼 표현에 사용하는 것은 혼란을 가져올 수 있다.

Kotlin에서 모든 연산자는 실제로 구체적인 이름을 가진 함수의 'syntactic sugar'에 지나지 않는다.  
아래 표와 같이 연산 문법을 사용하는 대신, 모든 연산자를 함수 호출로 대체할 수 있다.

| operator | Corresponding function |
|----------|------------------------|
| +a       | `a.unaryPlus()`        |
| -a       | `a.unaryMinus()`       |
| !a       | `a.not()`              |
| --a      | `a.dec()`              |
| a+b      | `a.plus(b)`            |
| a-b      | `a.minus(b)`           |
| a*b      | `a.times(b)`           |
| a/b      | `a.div(b)`             |
| a%b      | `a.rem(b)`             |
| a..b     | `a.rangeTo(b)`         |
| a in b   | `a.contains(b)`        |
| a += b   | `a.plusAssign(b)`      |
| a -= b   | `a.minusAssign(b)`     |
| a *= b   | `a.timesAssign(b)`     |
| a /= b   | `a.divAssign(b)`       |
| a %= b   | `a.remAssign(b)`       |
| a == b   | `a.equals(b)`          |
| a != b   | `a.notEquals(b)`       |
| a > b    | `a.compareTo(b) > 0`   |
| a < b    | `a.compareTo(b) < 0`   |
| a >= b   | `a.compareTo(b) >= 0`  |
| a <= b   | `a.compareTo(b) <= 0`  |
| a && b   | `a.and(b)`             |

Kotlin에서 각 연산자가 항상 일정한 의미를 갖도록 하는 것은 중요한 설계 원칙 중 하나이다.
예를 들어, 스칼라는 개발자에게 연산자 오버로딩을 무한하게 할 수 있도록 한다. 하지만, 이러한 자율성은 개발자들에 의해 잘못 사용될 수 있다.
특히, 낯선 라이브러리를 처음 사용할 때, 함수와 클래스의 이름에 명확한 의미가 존재하여도 
연산자가 특정 컨텍스트나 이론에만 익숙한 개발자들에게만 알려진 다른 의미로 사용되는 경우, 이는 코드를 이해하기 어렵게 만들 수 있다.

Kotlin에서는 각 연산자에 구체적인 의미를 부여함으로써 이런 문제를 방지한다.  
예를 들어 ` x + y == z`라는 표현식은 이것이 `x.plus(y).equals(z)`와 동일하다는 것을 의미한다.  
또한, `plus`가 'Nullable 타입'을 갖는다면 `x.plus(y))?.equals(z) ?: (z == null)`과 같음을 알 수 있다.

이처럼 명확한 이름을 갖는 연산자들은 예상대로 동작하도록 설계 하였으며, 이는 각 연산자가 사용될 수 있는 범위를 제한함으로써 코드를 이해하기 쉽게 만든다.  
앞서 예시로 말한 `not`을 사용하여 팩토리얼을 반환하는 것은 이 관례를 위반하는 명백한 행위로 절대 발생해서는 안된다.

---

## Unclear cases

가장 큰 문제점은 어떤 사용법이 기존의 'Convention'을 따르고 있는지 알기 어려울 때 발생한다.

예를 들어, '함수를 3배로 확장한다'는 것이 정확히 어떤 의미를 지니는지 생각해보면, 
몇몇 사람들은 주어진 함수를 3번 반복 실행하는 새로운 함수를 만드는 것이라고 생각할 수 있다.

```kotlin
operator fun Int.times(operation: () -> Unit): () -> Unit = {
    repeat(this) { operation() }
}

val tripledHello = 3 * { print("Hello") }
tripledHello() // HelloHelloHello
```

또 어떤 사람들은 주어진 함수를 3번 호출하려는 의도로 해석될 수 있다.

```kotlin
operator fun Int.times(operation: () -> Unit) {
    repeat(this) { operation() }
}

3 * { print("Hello") } // HelloHelloHello
```

의미가 명확하지 않을 경우, 확장 함수의 이름을 더 명시적으로 설명하듯 작성하는 것이 좋다.  
또한, 사용법을 연산자처럼 유지하고자 한다면, 이런 함수들을 'infix function'으로 정의할 수 있다.

```kotlin
infix fun Int.timesRepeated(operation: () -> Unit) = {
    repeat(this) { operation() }
}

val tripledHello = 3 timesRepeated { print("Hello") }
tripledHello() // HelloHelloHello
```

특정 경우에는, 최상위 함수를 사용하는 것이 더 바람직 할 수 있다.  
함수를 3번 반복하는 것은 이미 표준 라이브러리에 구현되어 제공되고 있다.

```kotlin
repeat(3) { println("Hello") } // HelloHelloHello
```

---

## When is it fine to break this rule

'DSL'을 설계하는 상황에서는 'Operator overloading'을 특이하게 사용하는 것이 허용됩니다.  
예를 들어, HTML을 위한 클래식한 DSL을 보면 다음과 같습니다.

```kotlin
body {
    div {
        +"Some Text"
    }
}
```

요소 안에 텍스트를 추가하는데 `String.unaryPlus` 메서드를 사용하는 것을 확인할 수 있다.  
이러한 사용법은 'DSL'의 일부이기에 허용되는 접근 방식이다.