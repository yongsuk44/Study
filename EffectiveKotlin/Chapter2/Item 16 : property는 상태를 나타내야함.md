# Item 16: Properties는 행동이 아닌 상태를 나타내야 합니다.

---

Kotlin의 `Property(이하 속성)`들은 Java의 `Field(이하 필드)`와 유사하게 보이지만, 실제로는 다른 개념을 나타냅니다.

```kotlin
//Kotlin property
var name : String?= null
```

```java
//Java field
String name = null;
```
데이터를 보유하는 데에 동일한 방식으로 사용될 수 있지만, 속성은 더 많은 기능을 가지고 있습니다.  
그 중, 사용자 정의 `setter`와 `getter`를 가질 수 있습니다.

```kotlin
var name: String? = null
    get() = field?.toUpperCase()
    set(value) = {
        if (!value.isNullOrBlank()) {
            field = value
        }
    }
```

## 백업 필드
컴파일러는 속성에 대한 백업 필드를 자동으로 생성하게 됩니다. 이 백업 필드는 `field` 키워드를 통해 접근할 수 있습니다.  
이러한 백업 필드는 `setter`와 `getter`의 기본 구현이 이를 사용하기 때문에 기본적으로 생성됩니다.

위 `name` 속성은 `field`라는 백업 필드를 가지게 됩니다.
`get()`에서 `field?.toUpperCase()`를 호출하면, 실제로는 백업 필드의 값을 대문자로 변환하여 반환하게 됩니다.
또한 `set(value)`에서는 입력 값 `value`가 null과 blank가 아닌 경우에만 백업 필드의 값을 변경합니다.
 
백업 필드를 사용하지 않는 사용자 정의 접근자를 구현 할 수 있으며, 그럴 경우 속성에는 전혀 필드가 없게 됩니다.   
예를 들어, 읽기 전용 속성 `val`에 대해서만 `getter`를 사용하여 Kotlin 속성을 정의할 수 있습니다.

```kotlin
val fullName : String
    get() = "$name $subname"
```

읽기-쓰기 속성 `var`의 경우, `getter`와 `setter`를 정의하여 속성을 만들 수 있습니다.

```kotlin
var dateVariable: Date
    get() = Date(millis)
    set(value) { millis = value.time }
```
이러한 속성들은 파생 속성(derived properties)으로 알려져 있습니다.

## 파생 속성
파생 속성은 다른 속성, 변수, 또는 연산에 기반한 속성을 말합니다. 
즉, 한 속성의 값이 다른 속성의 값에 의존하는 경우, 그 속성을 파생 속성이라고 합니다.

예를 들어, `fullName`이라는 속성이 `name`과 `subname`이라는 두 개의 다른 속성을 결합한 값을 반환하는 경우, `fullName`은 파생 속성이 됩니다.
`name`이나 `subname` 값이 변경되면, `fullName`도 그에 따라 변경됩니다.

#### 파생 속성의 장점

- 코드의 가독성과 개발자가 의도한 로직을 명확히 표현할 수 있습니다.
  - 파생 속성을 사용하면, 한 속성이 다른 속성에 어떻게 의존하는지를 코드로 명확히 표현할 수 있습니다.  

- 속성의 캡슐화가 간단하여 데이터의 안정과 유효성을 보장할 수 있습니다.
  - `fullName`을 직접 변경하게 아닌, `name`과 `subname` 값을 변경하는 방식으로 `fullName` 값을 간접적으로 제어할 수 있습니다.

- 쉽게 데이터 타입의 변경에 대응할 수 있습니다.
  - 예를 들어, 위 예제 코드에서 날짜 처리를 위해 `Date` 타입을 사용하다가 `Long` 타입으로 변경해야 하는 경우, 
  `dateVariable`이라는 파생 속성을 통해 `millis`라는 실제 데이터를 변환하여 제공함으로써, 
  `dateVariable`을 참조하는 외부 코드를 수정하지 않고도 변화에 대응할 수 있습니다.

## 속성 접근자
Kotlin의 속성은 필드가 아닌 접근자를 나타냅니다.   
이는 `val`이나 `var`에 대한 `getter` 혹은 `setter`를 의미합니다.   
이러한 속성의 설계는 `interface`에서도 이용될 수 있으며, `getter`의 존재를 약속합니다.

```kotlin
interface Person {
    val name: String
}
```

이런 설계로 인해 속성을 재정의할 수도 있습니다.   
예를 들어, 부모 클래스에서 정의한 속성을 자식 클래스에서 재정의할 수 있습니다.

```kotlin
open class SuperComputer {
    open val theAnswer: Long = 42
}

class AppleComputer : SuperComputer() {
    override val theAnswer: Long = 1_800_275_2275
}
```

또한, 속성의 값을 다른 객체에게 위임하는 것도 가능합니다. 이는 `by` 키워드를 통해 달성할 수 있습니다.

```kotlin
val db: DataBase by lazy { connectToDb() }
```

**속성이 본질적으로 함수**이기 때문에, 확장 속성(다른 클래스의 멤버처럼 보이는 속성을 추가하는 것)을 만들 수 있습니다.

```kotlin
val Context.preferences: SharedPreferences
    get() = PreferenceManager.getDefaultSharedPreferences(this)
```

이렇게 속성은 본질적으로 함수로, 그것을 어떻게 사용할지를 결정하는데 있어 주의가 필요합니다.   
적절한 상황과 사용 방법에 따라 코드의 간결성과 가독성을 향상시키는 도구로 작용할 수 있습니다.

## 속성 사용 시 주의점
속성은 아래 예제와 같이 **알고리즘적인 행동을 나타내는 데에 사용되어서는 안됩니다.**
즉, 속성이나 그 접근자가 복잡한 계산이나 상당한 시간이 소요되는 연산을 수행해서 안된다는 것을 의미 합니다.

```kotlin
// Don't do this
val Tree<Int>.sum: Int 
    get() = when (this) {
        is Node -> left.sum + right.sum
        is Leaf -> value
    }
```

여기서 `sum` 속성은 이진 트리의 모든 요소를 순회하므로 알고리즘적인 행동을 나타냅니다.   
이는 속성의 일반적인 사용 패턴과는 맞지 않습니다. 속성이나 `getter`는 일반적으로 빠르고 가벼운 연산을 수행해야 합니다.  
대규모 컬렉션에 대해 복잡한 계산을 수행하는 것은 `getter`의 예상되는 행동이 아니며, 
이렇게 작성 시 코드를 읽는 사람이 혼동을 느낄 수 있고, 성능에도 좋지 못합니다.

따라서 위 `sum`은 속성이 아니라 함수로 작성해야 합니다.

```kotlin

fun Tree<Int>.sum(): Int = when (this) {
    is Node -> left.sum() + right.sum()
    is Leaf -> value
}
```

## 속성 사용 시 일반적인 규칙
속성은 주로 상태를 나타내거나 설정하는 데 사용되어야 하며, 복잡한 로직이나 계산이 포함되어서는 안됩니다.

속성 대신 함수를 사용해야 하는 경우를 판단하는 방법은 속성에 만약 `get` 또는 `set` 접두어를 붙여야 하는 경우에 함수를 사용하는 것이 더 적절합니다.

속성 사용을 피해야하는 몇가지 상황은 다음과 같습니다.

- 비용이 많이 드는 연산
  - 사용자는 속성 사용이 시간이 많이 걸리지 않을 것으로 생각합니다. 
    따라서 계산 비용이 높거나, 계산 복잡도가 O(1)보다 큰 연산에는 함수를 사용하는 것이 더 바람직합니다.
- 비지니스 로직 포함
  - 속성은 주로 상태를 나타내는데 사용되므로, 복잡한 비지니스 로직을 포함하면 사용자에게 혼란을 줄 수 있습니다.
- 결과가 변하는 경우
  - 연속으로 같은 속성을 호출하였을 때 결과가 달라지면 안됩니다. 
    결과가 달라지는 경우 함수를 사용하는 것이 적절합니다.
- 실행순서가 중요한 경우
  - 속성은 어떤 순서로든 읽거나 쓸 수 있어야 합니다.
    순서가 중요한 경우에는 함수를 사용하는 것이 적절합니다.
- 형변환이 필요한 경우
  - 형변환은 일반적으로 메소드나 확장 함수를 통해 수행합니다. 
    속성을 사용하여 전체 객체를 변환하는 대신, 일부분만 참조하는것 처럼 보일 수 있습니다.
- Getter가 상태를 변경하는 경우
  - `getter`는 상태를 읽는데 사용되므로, 이를 호출하면 상태가 변경되어서는 안됩니다.