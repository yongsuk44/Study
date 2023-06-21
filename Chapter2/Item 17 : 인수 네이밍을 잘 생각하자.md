## Item 17 : 인수 네이밍 하는것을 고심해서 작성하자

---

코드를 읽을 때, 인수(argument)가 무엇을 의미하는지 항상 명확하지는 않습니다. 

다음 코드를 보시죠.

```kotlin
val text = (1..10).joinToString("|")
```

`jointToString`에 익숙한 개발자라면 `|`이 구분자(separator)라는것을 쉽게 알 수 있지만, 그렇지 않은 사람이라면 어떤 용도인지 정확하지 않습니다.

그래서 개발자들은 함수의 argument가 무엇을 의미하는지 명확하게 나타낼수 있도록 만들 수 있습니다.
가장 좋은 방법은 `named arguments(이하 명명된 인수)`를 사용하는 것입니다.

```kotlin
val text = (1..10).joinToString(separator = "|")
```

다른 방법으로 변수를 통해 사용할 수 있습니다.

```kotlin
val separator = "|"
val text = (1..10).joinToString(separator)
```

그러나 인수에 이름을 붙이는 것이 코드의 신뢰성을 더 높일 수 있습니다.  
변수 이름은 개발자의 의도가 명확하지만, 반드시 변수가 올바른 값인지는 보장하지 않습니다.

만약 개발자가 실수로 변수를 잘못된 위치에 놓았거나, 매개변수의 순서가 변경될 경우 예기치 못한 문제가 생길 수 있습니다.  
이러한 상황에서 명명된 인수는 이러한 문제를 방지할 수 있습니다.

따라서 우리는 값에 이름을 붙여도 명명된 인수를 사용하는 것이 더 합리적입니다.
```kotlin
val separator = "|"
val text = (1..10).joinToString(separator = separator)
```