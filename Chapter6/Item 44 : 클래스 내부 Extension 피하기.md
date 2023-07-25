# Item 44: 클래스 내부 Extension 피하기

클래스에 확장 함수 정의 시, 확장 함수는 첫 번째 인자로 받은 객체에 대한 호출을 수행하는 특별한 함수일 뿐 해당 **클래스의 멤버 함수가 아닙니다.**  
이러한 함수들은 내부적으로 일반 함수로 컴파일되며, 수신(receiver) 객체는 첫 번째 파라미터로 전달됩니다.

다음과 같은 확장 함수가 그렇습니다. 

```kotlin
fun String.isPhoneNumber(): Boolean = length == 7 && all { it.isDigit() }

// 위 코드를 컴파일하면 다음과 같이 변경
fun isPhoneNumber(`$this`: String): Boolean = $this.length == 7 && $this.all { it.isDigit() }
```

이런 방식의 컴파일로 인해 확장 함수는 멤버 함수처럼 동작하는 것처럼 보일 뿐만 아니라, 인터페이스나 추상 클래스 등에서도 정의할 수 있게 됩니다.

```kotlin
interface PhoneBook {
    fun String.isPhoneNumber(): Boolean
}

class Fizz: PhoneBook {
    override fun String.isPhoneNumber(): Boolean =
        length == 7 && all { it.isDigit() }
}
```

그러나 이런 형태로 멤버 확장 함수를 정의하는것이 가능하더라도 이러한 방식은 자제해야 합니다.  
특히 확장 함수를 클래스 멤버로 정의함으로써 가시성을 제한하려는 시도는 바람직하지 않습니다.

```kotlin
// Bad practice
class PhoneBookIncorrect {
    fun String.isPhoneNumber(): Boolean =
        length == 7 && all { it.isDigit() }
}
```

이러한 방식으로 확장 함수 정의 시, 실제로는 가시성이 제한되지 않지만, 확장 함수를 사용하는 것이 더 복잡해집니다.  
위 방식을 사용하려면 확장 수신자(즉, 확장 함수를 적용하려는 객체)와 디스패치 수신자(확장 함수를 정의한 클래스의 인스턴스)를 모두 제공해야 합니다.  
이는 `PhoneBookIncorrect().apply { "1234567890".test() }`와 같은 방식으로 사용해야 합니다.

따라서 확장 함수의 가시성을 제한하려면, 가시성 한정자를 사용하고 로컬에 배치하지 않아야 합니다. 
아래는 확장 함수의 가시성을 제한하는 올바른 방법입니다:

```kotlin
// 이렇게 사용하면 가시성을 제한하게 됨
class PhoneBookCorrect {
    // ...
}

private fun String.isPhoneNumber(): Boolean = length == 7 && all { it.isDigit() }
```

## 멤버 확장을 피해야하는 이유

### 1. 참조 지원이 없음

멤버 확장 함수는 정적으로 참조할 수 없습니다.

```kotlin
val ref = String::isPhoneNumber
val str = "1234567890"
val boundedRef = str::isPhoneNumber

val refX = PhoneBookIncorrect::isPhoneNumber // Error
val boundedRefX = PhoneBookIncorrect()::isPhoneNumber // Error
```

### 2. 두 수신자에 대한 암시적 접근에 대한 혼동

```kotlin
class A { val a = 10 }
class B { 
    val b = 10
    
    fun A.test() = a + b
}
```

### 3. 멤버 확장 함수에선 확장•디스패치 수신자 모두 접근할 수 있습니다. 이는 확장 함수가 상태를 변경 또는 참조 시 혼란을 야기
```kotlin
class A { /* ... */ }
class B { 
    // ...
    
    fun A.update() = ... // Shell is update A or B?
}
```

### 4. 주니어 개발자는 멤버 확장에 대한 개념이 어려워 이해하기 어려울 수 있음

요약하면, 멤버 확장을 사용하는 합당한 이유가 있다면 사용해도 괜찮습니다. 
그러나 일반적으로는 위에서 언급한 단점들을 인식하고 이를 피하는 것이 좋습니다. 
가시성을 제한하려면 가시성 한정자를 사용해야 하며, 단순히 클래스 내부에 확장 함수를 위치시키는 것으로는 그 사용을 제한할 수 없습니다.
