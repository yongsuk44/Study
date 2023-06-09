## Arguemnt와 State에 대한 기대값을 지정하자

### ✏️ 기대값을 명시하는 방법
1. `require` : 인수에 대한 기대값을 명시하는 방법
2. `check` : 상태에 대한 기대값을 명시하는 방법
3. `assert` : 어떤것이 참과 거짓인지 명시하는 방법, `JVM`-`Test Code`에서 사용
4. `Elvis`를 사용한 반환 또는 예외 처리  

```kotlin
// Part of Stack<T>
fun pop(num: Int = 1): List<T> { 
    require(num <= size) { "Cannot remove more elements than current size" } 
    check(isOpen) { "Cannot pop from closed stack" } 
    val collection = innerCollection ?: return emptyList() 
    val ret = collection.take(num)
    innerCollection = collection.drop(num) 
    assert(ret.size == num)
    return ret
}
```

---

### 👀 기대값 명시의 중요성
위와 같은 기대값을 명시하는 코드들은 다음과 같은 이점을 가질 수 있다.
1. 문서를 읽지 않는 프로그래머들에게도 기대치가 보인다.
2. 기대값에 만족되지 않으면, 함수는 예외를 던져서 예상치 못한 동작을 방지한다.
3. 자체적으로 코드에 대한 일정한 검사가 된다. 이러한 조건이 코드에서 검사되면 단위 테스트를 하는 필요성이 줄어든다.
4. 위에서 나열한 모든 검사는 스마트 캐스팅과 함께 작동하므로, 캐스팅을 적게 한다.

> Arguemnt 검사부터 시작해서 State 확인까지 왜 이러한 검사가 필요한지에 대해 살펴보자

--- 
 
## Arguments

함수와 `Argument`를 함께 정의할 때, **타입 시스템을 통해 표현할 수 없는 인수**에 대한 몇 가지 기대사항이 있다.   
몇 가지 예시를 보자 🥲

- 숫자의 팩토리얼을 계산할 때, 해당 숫자가 양의 정수여야 한다는 요구사항
```kotlin
fun factorial(n: Int): Long {
    require(n >= 0) { "Number must be non-negative" }
    return if (n <= 1) 1 else factorial(n - 1) * n
}
```

- 클러스터를 찾을 때, 포인트 목록이 비어있지 않아야 한다는 요구사항
```kotlin
fun findCluster(points: List<Point>): List<Cluster> {
    require(points.isNotEmpty()) { "At least one point required" }
}
```

- 사용자에게 이메일을 보낼 때, 해당 사용자가 이메일을 가져야 하며 이 값이 올바른 이메일 주소여야 한다는 요구사항
```kotlin
fun sendEmail(user: User) {
    require(user.email != null && user.email.matches(EMAIL_REGEX)) { "Email required" }
}
```

위처럼 요구사항을 명시하는 가장 범용적이고 직접적인 방법은 `require` 함수를 사용하는 것이다.   
이 함수는 해당 요구사항을 확인하고, 요구사항이 충족되지 않으면 예외를 `throw`하도록 한다.

---

## State
개발자들은 함수를 만들고 특정 조건에서만 실행할 수 있도록 개발한다.   
이러한 경우에 대한 몇 가지 일반적인 예시를 살펴보자 

- 일부 함수는 객체가 먼저 초기화되어야 하는 경우
```kotlin
fun speak(text: String) {
    check(isInitialized) { "TextToSpeech not initialized" }
}
```

- 사용자가 로그인한 경우에만 작업을 허용
```kotlin
fun getUserInfo(): UserInfo {
    check(isLoggedIn) { "User must be logged in" }
}
```

- 함수는 객체가 열려있어야 하는 경우
```kotlin
fun next(): T {
    check(isOpen) { "Cannot call next() on closed iterator" }
}
```

---

## Assertions

올바른 구현의 경우, 우리가 의도하여 구현하는 코드들이 있다.  

> 요청 : 함수에 10개의 요소를 반환  
> 기대 : 실행 시 10개의 요소가 반환

모든 함수들이 우리 의도대로 실행되면 참 좋겠지만, 그렇지 않은 경우가 많을 것이다.  
아마 구현을 잘못했을 수도 있고 함수가 리팩토링되어 제대로 작동하지 않을 수도 있다.
이러한 문제들을 해결하는 가장 범용적인 방법은 단위 테스트 코드를 작성하는 것이다.

```kotlin
@Test
fun `Stack pops Correct Number Of Elements`() {
    val stack = Stack<Int>(20) { it }
    val ret = stack.pop(10)
    assertEqulas(10, ret.size)
}
```

위처럼 단위 테스트는 구현의 정확성을 확인하는 범용적인 방법이지만,  
위와 같은 단위테스트는 **단일 사례에 대한 Stack에 관한 테스트**이다.  
`popRange()`에 대한 **검증**을 위해서는 아래와 같이 함수 내에서 `assert`를 사용하는것이 좋다.

```kotlin
fun pop(number: Int = 1): List<T> {
    // ... 
    assert(ret.size == number)
    return ret
}
```

### 함수 내에 `assert`를 포함하는 이점
- `assert`는 자체적으로 코드를 확인하고 효과적인 테스트 수행한다.
- 실행 지점에서 정확히 무언가를 확인하는 데 사용할 수 있으며 예상치 못한 동작이 시작된 위치를 쉽게 찾을 수 있다.

---

## Nullability과 Smart-Casting

`require`와 `check` 모두 [`Kotlin contracts`](../용어.md#contract)을 가지고 있으며, 이 `contracts`은 해당 함수가 반환될 때 해당 `predicate`가 참이라는 것을 명시한다.

```kotlin
public inline fun require(value: Boolean): Unit {
    contract { returns() implies value }
    require(value) { "Failed requirement." }
}
```
위와 같은 `require`에서 확인된 모든 것은 동일한 함수 내에서 나중에도 `true`로 취급된다.   
이는 **`Smart-Casting`과 잘 작동하는데, 한 번 확인된 것이 `true`라면 컴파일러가 그렇게 인식하기 때문이다.**

```kotlin
fun changeDress(person: Person) {
    require(person.outfit is Dress)
    val dress: Dress = person.outfit
}
```

`require`을 통해 `null`을 확인하려면 `requireNotNull()` `checkNotNull()`을 통해 확인할 수 있다.
```kotlin
data class User(val email: String?)

fun sendEmail(user: User) {
    requireNotNull(user.email) { "Email required" }
    validateEmail(user.email)
}
```

### Nullability

`Elvis 연산자`를 오른쪽에 `throw`나 `return`과 함께 사용하는 것이 일반적이다.   
이러한 구조는 높은 가독성과 더 많은 유연성을 제공한다. 

```kotlin
fun sendEmail(user: User) {
    val email = user.email ?: return
    validateEmail(email)
}

fun sendEmail(user: User) {
    val email = user.email ?: throw IllegalArgumentException("Email required")
    validateEmail(email)
}

fun sendEmail(user: User) {
    val email = user.email ?: run { Log.e("Email required") }
    return
}
```