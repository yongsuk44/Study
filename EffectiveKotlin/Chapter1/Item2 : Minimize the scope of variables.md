상태를 정의할 때, 변수와 프로퍼티의 범위를 좁히는 것이 좋다.

- 프로퍼티 대신 'local 변수' 사용
- 가능한 최소 범위의 변수 사용 (e.g: 변수가 loop 내에서만 사용된다면, loop 내에서 정의)

컴퓨터 프로그램에서 요소의 범위는 해당 요소가 보이는 영역으로,  
Kotlin에서는 범위가 거의 중괄호에 의해 생성되며, 일반적으로 외부 범위 요소에 접근할 수 있다. 

```kotlin
val a = 1
fun fizz() {
    val b = 2
    print(a + b)
}
val buzz = {
    val c = 3
    print(a + c)
}
// a를 모두 사용할 수 있지만, b와 c는 서로 사용할 수 없음.
```

위 예시에서 'fizz'와 'buzz' 함수의 범위 안에서는 외부 범위 변수에 접근이 가능하다.  
하지만, 외부 범위에서는 위 함수들 안에 정의된 변수에 접근이 불가능하다. 변수의 범위를 제한하는 방법에 대한 예시는 아래와 같다.

```kotlin
// Bad
var user: User
for (i in users.indices) {
    user = users[i]
    print("User at $i is $user")
}

// Better
for (i in users.indices) {
    val user = users[i]
    print("User at $i is $user")
}

// Same variables scope, nicer syntax
for ((i, user) in users.withIndex()) {
    print("User at $i is $user")
}
```

첫 번째 예제에서, `user` 변수는 'for-loop' 범위 안에서뿐만 아니라 그 외부에서도 접근이 가능하며,  
두 번째와 세 번째 예제에서 `user` 변수의 범위는 'for-loop' 범위에 구체적으로 제한되어 있다. 

위와 같이 '변수 범위'를 좁게 설정하면, 해당 변수가 영향을 미치는 코드의 양이 줄어든다.  
이는 코드를 이해하고 디버깅하기 더 쉬워지며, 변수가 전역적으로 접근 가능하거나 너무 넓은 범위에서 사용될 경우, 
변수가 어떻게 변경되고 사용되는지 추적하기 어려워 진다. 이러한 이유로 범위를 제한함으로써 코드 복잡성을 줄이고 오류 가능성을 낮출 수 있다. 
이는 프로퍼티나 객체를 가변보다 불변을 선호하는 이유와 유사하다.

가변 프로퍼티에 대해 생각해보면, 이들이 더 작은 범위에서만 수정이 가능하면 그 변화를 추적하기 용이하고, 이에 대한 추론과 동작의 변경이 더 쉬워진다.

넓은 범위의 변수의 문제 중 하나로 해당 변수를 프로그램의 다양한 부분에서 사용할 가능성이 있다는 문제가 있다. 
이는 변수 사용 목적이 모호해지고, 다른 개발자들이 해당 변수 목적을 추론해야 하는 문제를 초래한다. 
예를 들어, 'loop'에서 다음 요소를 할당하기 위해 사용되는 변수에, 어떤 개발자가 'loop' 이후에 마지막 요소를 사용하기 위해 만들었다고 오해 할 수 있다.
이와 같은 추론은 'loop' 이후 마지막 요소를 이용해 어떤 작업을 하는것과 같은 끔찍한 남용으로 이어질 수 있다.
이처럼 다른 개발자가 해당 변수에 어떤 값이 있는지 이해하기 위해 프로그램을 추론해야 하므로 불필요한 복잡성을 초래한다.

변수를 `val` 또는 `var` 중 어떤 것으로 선언하든, 개발자는 항상 변수가 정의될 때 초기화되는 것을 선호한다. 
개발자가 변수가 어디에서 정의되었는지 찾도록 강제하는 것은 좋지 않다. 
이는 `if`, `when`, `try-catch`, `?:`와 같은 제어 구조를 표현식으로 사용하여 지원할 수 있다.

```kotlin
// Bad
val user: User
if (hasValue) {
    user = getValue()
} else {
    user = User()
}

// Better
val user: User = 
    if (hasValue) {
        getValue()
    } else {
        User()
    }
```

만약, 여러 프로퍼티를 설정해야 한다면, '구조 분해 선언(destructuring declarations)'을 사용하자.

```kotlin
// Bad
fun updateWeather(degrees: Int) {
    val description: String
    val color: Int
    
    if (degrees < 5) {
        description = "cold"
        color = Color.BLUE
    } else if (degrees < 23) {
        description = "mild"
        color = Color.YELLOW
    } else {
        description = "hot"
        color = Color.RED
    }
}

// Better
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 5 -> "cold" to Color.BLUE
        degrees < 23 -> "mild" to Color.YELLOW
        else -> "hot" to Color.RED
    }
}
```

마지막으로, 너무 넓은 변수 범위는 위험할 수 있으며, 아래 일반적인 위험성을 알아보자.

---

## Capturing

'Sequence builder'를 사용하여 에라토스테네스의 체 알고리즘을 구현하여 소수를 찾는 방식은 다음과 같다.

1. 2부터 시작하는 숫자 목록을 가져온다.
2. 첫 번째 숫자를 가져온다. 이는 소수일 것이다.
3. 나머지 숫자들에서 첫 번째 숫자를 제거하고, 이 소수로 나눌 수 있는 모든 숫자를 필터링한다.

```kotlin
var numbers = (2..100).toList()
val primes = mutableListOf<Int>()

while (numbers.isNotEmpty()) {
    val prime = numbers.first()
    primes.add(prime)
    numbers = numbers.filter { it % prime != 0 }
}

println(primes)  // [2, 3, 5, 7, 11, 13, .... , 83, 89, 97]
```

위 코드를 아래와 같이 잠재적으로 무한한 소수의 시퀀스를 생성하는 코드를 작성할 수 있다.

```kotlin
val primes: Sequence<Int> = sequence {
    var numbers = generateSequence(2) { it + 1 }

    while (true) {
        val prime = numbers.first()
        yield(prime)
        numbers = numbers.drop(1).filter { it % prime != 0 }
    }
}

print(primes.take(10).toList()) // [2,3,5,7,11,13,17,19,23,29]
```

위 코드를 최적화 하려는 사람들은, 'loop'에서 매번 변수를 생성하지 않기 위해 소수를 가변 변수로 추출한다.

```kotlin
val primes: Sequence<Int> = sequence {
    var numbers = generateSequence(2) { it + 1 }

    var prime: Int
    while (true) {
        prime = numbers.first()
        yield(prime)
        numbers = numbers.drop(1).filter { it % prime != 0 }
    }
}

print(primes.take(10).toList()) // [2, 3, 5, 6, 7, 8, 9, 10, 11, 12]
```

출력 결과를 보면, 위 구현은 올바르게 동작되지 않는다.

왜냐하면, `prime` 변수를 캡처했기 때문이다. 필터링 작업은 시퀀스를 사용하기에 게으르게 수행된다. 
게으른 계산은 필터가 적용될 때가 아닌, 시퀀스 요소가 실제로 평가되어야 할 떄(`numbers.first()`) 연산을 수행한다.
따라서 `prime` 값이 변경될 때마다 이전 필터들도 새로운 `prime` 값에 영향을 받게된다.

```kotlin
// numbers = [ 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, ... ]

// Loop #1
prime = numbers.first() // numbers first '2'
yield(prime) // primes add '2'
// `drop(1).filter { ... } ` operations are lazy, so they are not executed yet 

// Loop #2
// 실제 numbers 시퀀스가 평가 되어야 할 때 == first(), toList(), take() 등이 호출될 때
prime = numbers
    .drop(1).filter { it % 2 != 0 } // drop '2' and filter '4', '6', '8', ...
    .first() // numbers first '3'

yield(prime) // primes add '3'
// `drop(1).filter { ... } ` operations are lazy, so they are not executed yet

// Loop #3
// `prime` 변수가 반복문 외부에 캡처되었기 때문에, 이전 prime 값에 영향을 받음
prime = numbers
    .drop(1).filter { it % 3 != 0 } // drop '2' and filter '3', '6', '9', ...
    .drop(1).filter { it % 3 != 0 } // drop '4'
    .first() // numbers first '5'

yield(prime) // primes add '5'
```

위와 같은 로직으로 인해 올바르게 소수를 필터링하지 못하고, 단순하게 숫자들을 걸러내는 결과를 얻게 된다.  
이와 같은 상황이 발생하기에 의도하지 않은 캡처에 대한 위험성을 알아야 하며, 이를 방지하기 위해 변수의 범위를 좁히는 것이 좋다.