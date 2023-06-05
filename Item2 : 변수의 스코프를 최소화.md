# 변수 스코프 최소화

---

변수의 스코프를 최소화하는 것은 좋은 프로그래밍 방법 중 하나가 될 수 있다.  
이는 변수를 최대한 좁은 범위에서 사용하는 것을 권장한다.   
특히, 반복문 내에서만 변수가 사용되는 경우 그 변수는 반복문 내에서 정의되어야 한다.

```kotlin
// 개선 전
var user: User
for (i in users.indices) {
  user = users[i]
  print("User at $i is $user")
}

// 개선 후
for ((i, user) in users.withIndex()) {
  print("User at $i is $user")
}
```
위 처럼 변수의 범위를 좁히면 개발자가 변수가 어디서 정의되고 사용되는지 쉽게 파악할 수 있기 때문에 프로그램의 추적이 쉬워지고 유지보수가 용이해진다.  
또한 변수는 **선언**될 때 항상 **초기화** 되는것이 좋다.

 
- 만약 여러 속성을 설정해야 할 경우 `destructuring decla-rations`(구조 분해 선언)을 활용하자

```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 5 -> "cold" to Color.BLUE
        degrees < 23 -> "mild" to Color.YELLOW
        else -> "hot" to Color.RED
    }
}
```

---

## 캡처링

> 람다 본문 안에서 외부 변수를 사용하는 행위

소수를 구하기 위해 시퀀스 빌더를 사용, 에라토스테네스의 체 구현

1. 2부터 시작하는 숫자 목록을 가져옴
2. 첫번 째 요소는 소수이므로 선택
3. 나머지 숫자에서 첫 번째 요소를 제거하고 이 소수로 나눌 수 있는 모든 숫자를 필터링

### 단순한 예제

```kotlin
var numbers = (2..100).toList()
val primes = mutableListOf<Int>()

while (numbers.isNotEmpty()) {
    val prime = numbers.first()
    primes.add(prime)
    numbers = numbers.filter { it % prime != 0 }
}

println(primes)  // [2,3,5,7,11,13, .... ,89,97]
```

### 시퀀스를 활용한 예제

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

### 시퀀스를 활용한 예제에서 리팩토링을 잘못한 예시

```kotlin
val primes: Sequence<Int> = sequence {
    var numbers = generateSequence(2) { it + 1 }

    var prime: Int
    while (true) {
        prime = numbers.first()
        yield(prime)
        numbers = numbers.filter { it % prime != 0 }
    }
}

print(primes.take(10).toList())[2, 3, 5, 6, 7, 8, 9, 10, 11, 12]
```

#### 소수가 출력이 되지 않는다. 무엇이 잘못되었는가?

첫 번째 코드에서는 `prime`을 `while` 안에서 지역적으로 선언되고,  
매 반복마다 새로운 값이 할당되며 매 반복마다 새로운 `prime` 값으로 필터링을 수행

두 번째 코드에서는 `prime`이 `while` 밖에서 선언되고,  
매 반복마다 업데이트되지만, 변수의 범위가 반복문을 넘어 확장되었음

> 시퀀스는 연산이 **지연**되는 특성이 있다.  
> 즉, 시퀀스의 요소에 대한 연산은 그 결과가 실제로 필요한 시점에 이르기 전 까지 수행되지 않는다.

따라서 `filter` 연산이 실제로 수행되는 시점에는 **`prime`의 값이 반복문이 종료된 후의 최종 값**이다.  
이로인해 `prime`이 2일 때 필터링된 4를 제외하면 `drop` 연산만 실행한 결과를 얻게 된 것이다.

---

## 정리

- 위와 같은 이유로 변수의 스코프를 최소화하며 코드를 작성해야 한다.
- 지역 변수에 대해서도 `var`보다는 `val`을 사용하자.
- 변수가 람다로 `캡처`된다는 사실을 항상 인식해야 한다.