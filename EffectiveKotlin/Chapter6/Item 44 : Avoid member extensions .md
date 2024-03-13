어떤 클래스에 대해 확장 함수를 정의하더라도, 이 확장 함수는 클래스의 멤버로 추가되지 않는다.  
확장 함수는 'receiver'라고 하는 첫 번재 인자에 대해서 호출되는 특별한 타입의 함수이다.  
실제로, 확장 함수는 일반 함수로 변환되어 컴파일되며, 이 때 receiver는 함수의 첫 번째 파라미터로 처리된다.  

예를 들어, 다음 함수를 생각해보자. 

```kotlin
fun String.isPhoneNumber(): Boolean = 
    length == 7 && all { it.isDigit() }
```

내부적으로는 다음과 유사한 함수로 컴파일 된다.

```kotlin
fun isPhoneNumber(`$this`: String): Boolean = 
    $this.length == 7 && $this.all { it.isDigit() }
```

확장 함수의 구현 방식에 따른 결과 중 하나는, 멤버 확장을 만들 수 있다는 것이다.  
또한, 인터페이스 내에서 확장 함수를 정의하는 것도 가능하다.

```kotlin
interface PhoneBook {
    fun String.isPhoneNumber(): Boolean
}

class Fizz: PhoneBook {
    override fun String.isPhoneNumber(): Boolean =
        length == 7 && all { it.isDigit() }
}
```

멤버 확장을 정의할 수 있지만, 이를 피해야 하는 몇 가지 이유가 있다. (DSLs의 경우는 예외)  
특히, 'Visibility'를 제한하고자 한다면, 멤버 확장을 정의하지 않는 것이 좋다.

```kotlin
// Bad practice, Don't do this
class PhoneBookIncorrect {
    // ...
    
    fun String.isPhoneNumber(): Boolean =
        length == 7 && all { it.isDigit() }
}
```

가장 큰 이유 중 하나는 실제로 'visibility'를 제한하지 않기 때문이다.  
사용자가 확장 함수를 사용하기 위해, 확장 수신자와 디스패치 수신자를 모두 제공해야 하기 때문에, 확장 함수의 사용을 더 복잡하게 만든다.

```kotlin
PhoneBookIncorrect().apply { "1234567890".test() }
```

확장의 'visibility'를 제한하고자 한다면, 이를 로컬로 배치하는 것이 아닌, 'visibility modifier'를 사용하는 것이 바람직하다. 

```kotlin
// This is how we limit extension functions visibility
class PhoneBookCorrect {
    // ...
}

private fun String.isPhoneNumber(): Boolean = 
    length == 7 && all { it.isDigit() }
```

멤버 확장 사용을 피해야 하는 몇 가지 주요 이유는 다음과 같다. 

1. 참조가 지원되지 않는 문제가 있다.

```kotlin
val ref = String::isPhoneNumber
val str = "1234567890"
val boundedRef = str::isPhoneNumber

val refX = PhoneBookIncorrect::isPhoneNumber            // Error
val boundedRefX = PhoneBookIncorrect()::isPhoneNumber   // Error
```

2. 확장 수신자와 디스패치 수신자에 대한 암시적 접근이 혼란스러울 수 있다.

```kotlin
class A { 
    val a = 10
}

class B { 
    val b = 10
    
    fun A.test() = a + b
}
```

3. 확장이 수신자를 변경하거나 참조해야 할 때, 확장 수신자를 수정하는 것인지, 아니면 확장이 정의된 클래스인 디스패치 수신자를 수정하는 것인지 명확하지 않다.

```kotlin
class A {
    // ...
}

class B {
    // ...
    
    fun A.update() = ...      // Shell is update A or B?
}
```

4. 경험이 부족한 개발자들에게는 멤버 확장의 사용이 직관적이지 않거나 이해하기 어려울 수 있다.

결론적으로, 멤버 확장을 사용할 타당할 이유가 있다면 사용하는 것은 문제가 되지 않는다.  
그러나, 가능한 한 이를 피하려고 놀력하고, 멤버 확장의 단점을 잘 인지하고 있어야 한다.

'visibility'를 제한하고 싶다면, 'visibility modifier'를 적절히 사용해야 한다.  
단순히 확장 기능을 클래스 내부에 위치시킨다고 해서, 그것이 외부에서 사용되지 않도록 제한되지는 않는다.