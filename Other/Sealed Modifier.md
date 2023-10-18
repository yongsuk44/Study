# Sealed Modifier

> - `sealed` 키워드는 모듈 내로 상속을 제한하고 컴파일 시점에서 하위 타입을 알 수 있게 됨  
> - `sealed class`는 N개의 인스턴스와 상태 변경 O, `enum clss`는 단일 인스턴스와 상태 변경 X 
> - `class`는 단일 상속만 가능하기에 다중 상속이 필요하면 `interface` 사용

일반적인 클래스나 인터페이스는 어디에서든 상속이나 구현이 가능하지만,  
`sealed` 키워드가 붙은 `sealed class`와 `sealed interface`는 상속을 제어할 수 있도록 설계되었습니다.

이들의 직접적인 모든 하위 클래스는 컴파일 시점에서 이미 결정되어, 하위 클래스의 추가를 방지할 수 있습니다.  
이는 `sealed class`가 정의된 모듈이나 패키지 외부에서 추가적인 하위 클래스를 정의할 수 없다는 것을 의미합니다.  
이로 인해 `sealed class`의 인스턴스는 컴파일 시점에 알려진, 제한된 하위 클래스 중 하나에만 속하게 됩니다.

`sealed interface`도 `sealed class`와 비슷한 원칙을 따릅니다.  
`sealed interface`도 모듈이 컴파일된 후에는 새로운 구현을 추가할 수 없습니다.  
이로 인해 `sealed interface`를 사용하면서도 구현에 대한 제어를 더욱 강화하고, 예상치 못한 변경이나 확장을 방지할 수 있습니다. 

### 예시

예를 들어 라이브러리 개발자가 `Error` 클래스의 계층 구조를 제한하여 외부에서 새로운 오류 클래스를 생성하지 못하도록 할 수 있습니다.  
따라서 라이브러리 개발자는 라이브러리 내에서 정의한 오류 타입만을 고려하여 로직을 구현할 수 있습니다.

```kotlin
sealed interface Error
sealed class IOError(): Error

class FileReadError(val file: File): IOError()
class DatabaseError(val source: DataSource): IOError()

object RunTimeError : Error
```

---

## Sealed class vs Enum class

`sealed class`와 `Enum class`는 모두 제한된 타입의 집합을 가진다는 점이 유사합니다. 

하지만, `Enum class`의 각 상수는 단일 인스턴스만을 가지며, 그 상태는 변경될 수 없습니다.  
반면에, `sealed class`의 하위 클래스는 여러 인스턴스를 가질 수 있고, 각 인스턴스는 자신만의 상태를 가질 수 있습니다.

이러한 차이점으로 `sealed class`를 더 다양한 상황에서 유용하게 사용할 수 있습니다.

```kotlin
enum class UiState {
    Success, Error, Loading
}

sealed class RandomState {
    object Loading: RandomState()
    data class Success(val data: Int, val randomIndex: Int): RandomState()
    data class Error(val message: String): RandomState()
}
```

---

## Sealed class visibility

`sealed class`는 기본적으로 `abstract class`이므로, 직접 인스턴스화 할 수 없습니다.  
이는 오로지 하위 클래스를 통해서만 인스턴스화 될 수 있음을 의미합니다.   
또한 `abstract method` or `abstract properties`를 가질 수 있습니다.

`sealed class`의 생성자는 `protected` or `private` 가시성을 가질 수 있습니다.  
기본적으로 `protected`로 설정되어 있으며, 이는 `sealed class`가 같은 패키지(모듈)나 하위 클래스에서만 접근 가능함을 의미합니다.

```kotlin
sealed class IOError {
    constructor() { /*...*/ } // protected by default
    private constructor(description: String): this() { /*...*/ } // private is OK
    // public constructor(code: Int): this() { /*...*/ } // public and internal are not allowed
}
```

---

## Location of direct subclasses

`sealed class`와 `sealed interface`의 직접적인 하위 타입은 반드시 같은 패키지에 있어야 하기에, 라이브러리나 모듈을 작성할 때 이를 고려해야 합니다.

하지만, 이 하위 타입들이 어디에 선언되거나 어떤 가시성을 가질 것인지에 대해서는 일반 클래스 or 일반 인터페이스와 같은 규칙이 적용됩니다.  
즉, 일반 클래스처럼 다른 클래스 안에 중첩될 수 있고, 가시성(`private`, `protected`, `public`, `internal` 등)을 가지며 접근을 제한할 수 있습니다.

`sealed` 키워드는 직접적인 하위 타입에만 특별한 제약을 거는 것이고, 그 하위 타입이 또 다른 하위 타입에는 적용되지 않습니다.  
즉, 직접적인 하위 타입이 `open`과 허용되어 있다면, 어떤 방식으로든 확장이 될 수 있습니다.

```kotlin
sealed interface Error // has implementations only in same package and module
sealed class IOError(): Error // extended only in same package and module
open class CustomError(): Error // can be extended wherever it's visible
```

---

## Sealed classes and when expression

`sealed class`는 미리 정의된 하위 클래스만 가질 수 있기에, `when` 조건문과 같이 사용하면 컴파일러가 자동으로 모든 조건을 만족한다고 판단합니다.   
따라서 `else` 절을 추가할 필요가 없게되어 코드의 안전성을 높이고 버그 가능성을 줄여줍니다.

```kotlin
fun logError(e: Error) = when(e) {
    is DatabaseError -> println("DatabaseError")
    is FileReadError -> println("FileReadError")
    is RunTimeError -> println("RunTimeError")
}
```