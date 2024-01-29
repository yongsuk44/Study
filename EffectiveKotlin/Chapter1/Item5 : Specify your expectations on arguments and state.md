'기대사항(expectations)'이 있다면 가능한 먼저 명시하는 것이 좋다. 
이를 위해 Kotlin에서는 주로 아래 방법들을 사용한다.

- `require` block : 'argument'에 대한 기대사항을 명시하는 보편적인 방법
- `check` block : 'state'에 대한 기대사항을 명시하는 보편적인 방법
- `assert` block : 무언가가 'true'인지 확인하는 방법, JVM 환경의 테스트 모드에서 사용
- `return`이나 `throw`와 함께 사용하는 'Elvis 연산' 

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

위와 같은 방식으로 코드 내에서 특정 조건이나 기대사항을 명시하는 것이 전통적인 문서화 방법을 대체하지는 않지만, 매우 유용하다.   
이러한 선언적 검사('Declarative checks')는 아래와 같은 장점을 가지고 있다.

- 위 검사 방법들은 문서화를 완전히 대체할 수 있진 않지만, 코드 자체에서 기대사항을 분명히 함으로써 문서를 꼼꼼하게 읽지 않는 개발자들도 기대사항을 알 수 있다.
- 기대사항이 만족되지 않으면 예외를 발생 시킴으로써, 잠재적인 버그나 예상치 못한 행동으로 이어질 수 있는 문제들을 미리 차단한다.
특히, 상태가 변경되기 전에 예외가 발생되어 부분적인 상태 변경으로 인한 복잡한 문제를 방지할 수 있고, 이로 인해 상태 안정성을 높일 수 있다.
- 위 검사들로 자체적으로 코드가 일정 부분, 조건을 확인하고 유효성을 검증한다. 이는 'Unit Test'의 필요성을 줄여준다.
- 위 나열된 모든 검사 방법들은 'Smart-Casting'과 함께 잘 작동하여, 명시적인 타입 캐스팅의 필요성을 줄여준다.

--- 
 
## Arguments

함수에 'argument'를 정의할 때, '타입 시스템'을 사용하여 표현할 수 없는 'argument'에 대한 기대사항을 갖는 것은 일반적인 경우이다.
이와 같은 기대사항을 명시하는 보편적인 방법은 `require`를 사용하는 것으로, 요구사항을 검사하고 만족되지 않을 경우 예외를 던진다.
아래는 몇가지 에시이다.

- 숫자의 팩토리얼을 계산할 때, 해당 숫자가 양의 정수여야 할 수 있다.

```kotlin
fun factorial(n: Int): Long {
    require(n >= 0) { "Number must be non-negative" }
    return if (n <= 1) 1 else factorial(n - 1) * n
}
```

- 클러스터를 찾을 때, 포인트 목록이 비어있지 않아야 한다.

```kotlin
fun findCluster(points: List<Point>): List<Cluster> {
    require(points.isNotEmpty()) { "At least one point required" }
    // ...
}
```

- 사용자에게 이메일을 보낼 때, 해당 사용자가 이메일을 가지고 있고, 그 값이 올바른 이메일 주소여야 한다.

```kotlin
fun sendEmail(user: User, title: String, message: String) {
    requireNotNull(user.email) { "User must have email" }
    require(isValidEmail(user.email)) { "Email must be valid" }
    // ...
}
```

이러한 요구사항들은 함수의 맨 처음 선언되기에, 명확한 요구사항을 전달 할 수 있다.

`require`은 주어진 조건이 만족되지 않으면 `IllegalArgumentException`을 던지기에 명시된 기대사항을 무시할 수 없다.
또한 `require`이 함수 시작 부분에 위치함으로써, 'argument'가 잘못되었을 경우 함수가 즉시 중단된다. 
이는 잘못된 'argument'로 인해 발생할 수 있는 문제가 함수 내부 깊숙이 전파되는 것을 방지한다.
결과적으로, 함수 시작 부분에서 'argument'에 대한 기대사항을 명확히 설정함으로써, 해당 함수가 기대사항을 충족하는 'argument'로 호출된 것이라고 가정 할 수 있다.

---

## State

특정 조건에서만 일부 함수를 사용할 수 있도록 하는 것은 드문 일이 아니다.
이러한 경우에 대한 몇 가지 일반적인 예시를 살펴보자.

- 객체가 먼저 초기화 되어야 하는 경우

```kotlin
fun speak(text: String) {
    check(isInitialized) { "TextToSpeech not initialized" }
}
```

- 사용자가 로그인한 경우에만 특정 작업을 허용

```kotlin
fun getUserInfo(): UserInfo {
    check(isLoggedIn) { "User must be logged in" }
}
```

- 파일이나 데이터베이스 연결과 같이 객체가 '열린' 상태인 경우 

```kotlin
fun next(): T {
    check(isOpen) { "Cannot call next()" }
}
```

`check`는 '상태'가 올바른지 확인하며, 명시된 기대사항이 충족되지 않을 때 `IllegalStateException`을 던진다.
예외 메시지는 `require`과 마찬가지로 'lazy message'를 사용하여 커스터마이징 할 수 있다.
함수의 전체적인 상태 기대사항에 대해 `check`를 사용할 때는, 일반적으로 `require` 바로 뒤, 함수의 시작 부분에 배치한다.
일부 상태 기대사항은 특정 로컬 상황 또는 조건에만 적용될 수 있다. 이런 경우, `check`는 함수의 뒷부분에서도 사용할 수 있다.

---

## Assertions

함수에 10개 요소를 반환하도록 요청하면, 그 함수가 10개의 요소를 반환하도록 구현했을 때 
정확한 구현이라면 10개가 반환 되는것을 기대 하겠지만, 항상 바라는 것처럼 동작하지 않을 수 있다.
이는 구현에서 실수를 했을 수 있고, 사용한 함수가 변경되었거나 리팩토링 작업 등의 이유가 있을 것이다.
이러한 모든 문제들에 대한 가장 보편적인 해결책은 'Unit Test'를 작성하는 것이다.

```kotlin
@Test
fun `Stack pops correct number of elements`() {
    val stack = Stack<Int>(20) { it }
    val ret = stack.pop(10)
    assertEqulas(10, ret.size)
}
```

기본적으로, 'Unit Test'는 소프트웨어의 특정 부분이 예상대로 동작하는지 확인하는데 사용된다. 
'Stack'에서 요소를 제거하는 `pop`은 단순히 한번의 사용 사례에 대해서만 테스트를 하는 것이 아니라, 거의 모든 사용 사례에 대해 해당 기능의 정확성을 검증해야 한다.
즉, 개발자는 특정 기능의 가능한 모든 사용 사례를 고려하고, 그 중에서도 'edge-case'를 테스트 해야 한다.
위 상황에서는 `pop` 함수 내부에 'assertion'을 포함하여 모든 사용 사례에 정확성을 검증 하는 것이 좋을 것이다.

```kotlin
fun pop(number: Int = 1): List<T> {
    // ... 
    assert(ret.size == number)
    return ret
}
```

기본적으로 'assertion'은 Kotlin/JVM에서만 활성화 되어있고, JVM의 -ea(enable assertions) 옵션을 사용하여 명시적으로 활성화 하지 않는 한 검사되지 않는다.
그렇기에 'assertion'을 'Unit Test'의 일부로 간주해야 한다. 또한 기본적으로 테스트를 실행할 때만 기본적으로 활성화 되기에 'production 환경'에서 어떠한 오류도 발생시키지 않는다.
만약 심각한 오류가 발생할 가능성이 있다면 `check`를 사용해야 한다.

'Unit Test' 함수 내부에 `assert`를 포함시키는 것은 다음 장점을 가지고 있다.

- `assert`는 코드를 자가 검사하게 만들고, 더 효과적인 테스트를 가능하게 함.
- 특정 사례에 대한 것이 아닌, 실제 사용 사례에 대한 기대사항을 검사함.
- 실행의 정확한 지점에서 특정 조건이나 상태를 검증할 수 있게 해줌.
- 코드를 더 일찍 실패하게 만들어, 실제 문제에 더 가깝게 만들어줌으로써 문제의 원인을 더 빠르게 파악하고 해결 할 수 있게 해줌.

---

## Nullability and smart-casting

`contract`를 통해 `require`와 `check`는 검사 후 반환이 되면 그들의 조건이 '참'이라는 것을 명시한다. 

```kotlin
public inline fun require(value: Boolean): Unit {
    contract { returns() implies value }
    require(value) { "Failed requirement." }
}
```

`require` or `check`에서 검사된 모든 것들은 동일한 함수 내에서 '참'으로 간주된다.
이는 'smart-casting'과 잘 작동하는데, 한번 검사된 것이 '참'이라면 컴파일러가 그렇게 인식하기 때문이다.
아래 예제는 특정 프러퍼티 (`outfit`)이 특정 타입(`Dress`)인지 검사하고, 
이 검사를 통과하면 컴파일러는 `outfit`이 `Dress` 타입임을 알고, 그 이후 코드에서 `outfit`을 `Dress` 타입으로 취급한다.

```kotlin
fun changeDress(person: Person) {
    require(person.outfit is Dress)
    val dress: Dress = person.outfit
}
```

이런 독특함은 어떤 것이 `null`인지 확인할 때 유용하다.

```kotlin
class Person(val email: String?)

fun sendEmail(person: Person, title: String, text: String) {
    require(person.email != null)
    val email: String = person.email
}
```

위와 같이 `null` 확인을 위해 특별한 함수인 `requireNotNull()`과 `checkNotNull()`가 있다.  
두 함수 모두 변수에 대해 'smart-casting' 할 수 있는 능력을 가지고 있으며, 이를 'unpack' 표현으로도 사용할 수 있다.

```kotlin
class Person(val email: String?)
fun validateEmail(email: String) { /* ... */ }

fun sendEmail(person: Person, title: String, text: String) {
    val email = requireNotNull(person.email)
    validateEmail(email)
    // ...
}

fun sendEmail(person: Person, title: String, text: String) {
    requireNotNull(person.email)
    validateEmail(person.email)
    // ...
}
```

'Nullability'에 대해서 `throw` or `return`의 오른쪽에 Elvis 연산자(`?:`)를 사용하는 것은 좋은 방법이다.
이런 구조는 높은 가독성과 더 많은 유연성을 제공한다. 아래 예제는 오류를 발생시키는 대신 `return`을 통해 함수를 쉽게 중단하는 방법이다.

```kotlin
class Person(val email: String?)
fun sendEmail(person: Person) {
    val email: String = person.email ?: return
}
```

만약 프로퍼티가 잘못되어 'null'인 경우 추가적인 동작을 수행할 때, `return` or `throw`를 `run`으로 감싸서 사용할 수 있다.  
이 방법은 함수가 중단된 이유를 로그로 남겨야 하는 경우에 유용할 수 있다.

```kotlin
fun sendEmail(person: Person) {
    val email = user.email ?: run { 
        Log.e("Email required")
        return
    }
}
```