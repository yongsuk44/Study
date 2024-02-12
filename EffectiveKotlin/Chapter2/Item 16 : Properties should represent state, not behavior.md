Kotlin의 프로퍼티는 Java의 필드와 유사하게 보이지만, 실제로는 다른 개념을 나타낸다.

```kotlin
//Kotlin property
var name : String? = null
```

```java
//Java field
String name = null;
```

데이터를 저장하는데 있어서 같은 방식으로 사용될 수 있지만, 프로퍼티는 더 많은 기능을 가지고 있다.  
그 시작점 중 하나로, 프로퍼티는 'custom setter / custom getter'를 가질 수 있다.

```kotlin
var name: String? = null
    get() = field?.uppercase()
    set(value) {
        if (!value.isNullOrBlank()) {
            field = value
        }
    }
```

위 코드를 보면 'setter'에서 'field' 식별자를 사용하는데, 이는 해당 프로퍼티에 데이터를 저장할 수 있게 해주는 'Backing field'를 참조한다. 

'Backing field'는 프로퍼티에 값을 저장하기 위해 숨겨진 필드로, Kotlin 컴파일러는 프로퍼티의 'getter / setter'가 값에 접근할 때 이 'Backing field'를 자동으로 생성한다.
즉, 'field' 식별자는 프로퍼티의 'custom accessor' 내에서만 사용되며, 프로퍼티의 값을 읽거나 쓰는데 사용된다.  

만약 'custom accessors'가 'field'를 사용하지 않는 경우, Kotlin 컴파일러는 'Backing field'를 생성하지 않으며, 
이는 프로퍼티가 직접 값을 저장하지 않고, 다른 방식으로 값을 제공하거나 계산되어 사용됨을 의미한다.

예를 들어, 'Read-only val'의 프로퍼티에서 'getter'만 사용하여 정의할 수 있다.

```kotlin
val fullName : String
    get() = "$name $subname"
```

또한, 'Read-write var'의 경우, 'getter / setter'를 정의하여 프로퍼티를 만들 수 있다.

```kotlin
var dateVariable: Date
    get() = Date(millis)
    set(value) { 
        millis = value.time
    }
```

위와 같은 프로퍼티들을 'Derived properties'라고 하며, 해당 프로퍼티들이 Kotlin에서 모든 프로퍼티가 기본적으로 캡슐화되는 주된 이유이다.

Java 표준 라이브러리 'Date'를 사용하여 어떤 객체 내에 날짜를 저장해야 한다고 가정해보자.  
그런데 직렬화의 이유로 객체가 더 이상 해당 타입의 프로퍼티를 저장할 수 없게 되었다.  
만약 이때, 해당 프로퍼티가 프로젝트 전체에서 참조되고 있다면, 이를 수정하기 까다로워진다는 문제점이 발생하게 된다. 

하지만, Kotlin에서는 이러한 문제점이 발생하지 않는다.  
왜냐하면 데이터를 별도의 프로퍼티 'millis'로 이동시키고, 프로퍼티가 데이터를 저장하는 대신 'wrapper / un-wrapper' 역할을 하도록 수정할 수 있기 때문이다.

Kotlin에서 프로퍼티는 클래스의 데이터를 나타내는 주요 방식 중 하나이다. 하지만, 모든 프로퍼티가 'Backing field'를 갖는 것은 아니다.  
실제로, **프로퍼티는 데이터에 접근하는 방법을 나타내며**, 이러한 특성으로 프로퍼티는 상태를 저장할 수 없는 인터페이스에서도 사용될 수 있다.

아래와 같이 인터페이스를 구현하는 경우, 구현체는 해당 프로퍼티에 대한 'getter'를 제공해야 한다.

```kotlin
interface Person {
    val name: String
}
```

추가로, 앞서 프로퍼티는 데이터에 접근하는 방법이라고 설명하였다.  
이러한 프로퍼티 시스템의 설계 원칙은 아래와 같이 프로퍼티를 'override'하여 사용 할 수 있도록 한다. 

```kotlin
open class SuperComputer {
    open val theAnswer: Long = 42
}

class AppleComputer: Supercomputer() {
    override val theAnswer: Long = 1_800_275_2275
}
```

또한 동일한 이유로, 'property delegation'이 가능하다.

'Property delegation'은 반복되는 프로퍼티 접근 패턴을 추출하고 재사용할 수 있게 해준다.  
이는 아래와 같이 'by' 키워드를 사용하여 다른 객체에게 프로퍼티의 접근을 위임할 수 있게 해준다.

```kotlin
val db: DataBase by lazy { connectToDb() }
```

프로퍼티 접근 시 'setter / getter' 메서드가 호출되는데, 이러한 구조는 프로퍼티에 값을 할당하거나 값을 읽는 동작이 결국 함수 호출로 이루어짐을 의미한다.  
이처럼 프로퍼티는 본질적으로 함수라는 점을 고려할 떄, 아래와 같이 'Extension property'를 만들 수 있다.

```kotlin
val Context.preferences: SharedPreferences
    get() = PreferenceManager.getDefaultSharedPreferences(this)

val Context.inflater: LayoutInflater
    get() = getSystemService(Context.LAYOUT_INFLATER_SERVICE) as LayoutInflater

val Context.notificationManager: NotificationManager
    get() = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
```

프로퍼티는 'Java의 Field'가 아니라 'accessor'를 나타낸다는 것을 고려할 떄, 프로퍼티는 특정 함수들을 대체하여 사용될 수 있다.  
하지만, 프로퍼티는 아래 예시와 같이 알고리즘적 행동을 나타내는데 사용되어서는 안된다.

```kotlin
// Don't do this
val Tree<Int>.sum: Int 
    get() = when (this) {
        is Leaf -> value
        is Node -> left.sum + right.sum
    }
```

위 코드에서 'sum' 프로퍼티는 모든 요소를 순회하여 합을 계산하기에 대규모 컬렉션에서는 이 직업이 상당한 시간을 소모할 수 있다.  
또한 이러한 계산이 'getter' 내부에서 수행됨을 개발자가 명확하게 인지하기 어려울 수 있다.

이러한 이유로 프로퍼티의 'accessor'들은 간단한 변경 작업에 사용되어야 하며, 위와 같은 알고리즘적 행동을 나타낼 때에는 함수를 사용하는 것이 더 적절하다.
함수는 계산이나 로직 처리를 명시적으로 보여주며, 개발자들이 더 명확하게 구현 의도와 로직을 파악할 수 있게 해준다.

```kotlin
fun Tree<Int>.sum(): Int = when (this) {
    is Leaf -> value
    is Node -> left.sum() + right.sum()
}
```

프로퍼티를 사용하는 일반적인 규칙은 상태를 나타내거나 설정하는 것에만 한정되어야 하고, 그 외의 다른 로직은 포함되어서는 안된다.  
프로퍼티로 정의할지, 함수로 정의할지 판단하는 기준은 "만약 이를 함수로 정의한다면, 함수 이름 앞에 'get/set'을 붙일 것인가?" 이다.  
만약 'get/set'을 붙여야 한다면 프로퍼티로 정의하는 것이 좋고, 그렇지 않다면 함수로 정의하는 것이 더 적절하다.

보다 구체적으로, 아래 상황은 프로퍼티 대신 함수를 사용하는 것이 적절한 일반적인 상황이다.

**1. 계산적으로 비용이 많이 드는 연산이거나, '계산 복잡도 O(1)'보다 높은 경우** :  
비용이 많이 발생하는 연산이 프로퍼티로 정의되어 있다면, 이는 함수로 정의하는 것이 더 적절하다.  
왜냐하면 함수는 명시적으로 개발자에게 비용이 많이 발생할 것이라는 것을 알려줄 수 있고, 이를 덜 사용하거나, 캐싱을 고려하게 할 수 있기 때문이다.

**2. 비지니스 로직을 포함하는 경우** :  
개발자들이 코드를 읽을 때, 프로퍼티는 '로깅', '리스너 알림', '바인딩된 요소 업데이트'와 같은 간단한 동작을 수행한다고 예상한다.  
그 이상의 복잡한 비지니스 로직은 함수로 정의하는 것이 더 적절하다.

**3. 연속으로 멤버를 호출했을 때 다른 결과를 반환하는 경우** :  
프로퍼티나 함수를 호출할 때 일관된 결과를 기대하는 것은 프로그래밍에서 일반적인 원칙이다.  
그 중 특히, 프로퍼티는 동일한 입력에 대해 항상 동일한 출력을 반환하는 것이 일반적인 원칙이다.  
그러나 어떤 프로퍼티의 값이 외부 상태에 의존하거나 내부 상태를 변경함으로써, 연속으로 호출할 때 다른 결과를 생성하면, 이는 프로퍼티 사용의 부적절한 사례이다.  
이처럼, 연속적으로 호출했을 때 다른 결과를 반환하는 로직의 경우에는 함수로 정의하는 것이 더 적절하다.

**4. 실행 순서가 중요한 경우** :  
코드에서 프로퍼티의 값이 설정되거나 검색되는 순서가 앱의 동작에 영향을 미치는 경우, 이는 프로퍼티 사용의 부적절한 사례이다.  
프로퍼티는 독립적으로 동작하며, 하나의 프로퍼티 설정이 다른 프로퍼티의 값이나 동작에 직접적인 영향을 미치지 않는 것이 기본 원칙이다.  
이처럼, 프로퍼티의 설정과 검색 순서가 앱의 동작에 영향을 미치는 경우, 이러한 로직은 명시적으로 메서드로 관리하는 것이 더 적절하다.

**5. 'Int.toDouble()'과 같이 타입 변환이 필요한 경우** :  
Kotlin에서 타입 변환은 일반적으로 메서드 또는 확장 함수를 통해 수행되는 것이 'Convention'이다.   
만약 프로퍼티를 통해 타입 변환을 구현하게 되면, 객체 내부 상태의 단순한 접근처럼 보일 수 있으므로 개발자에게 혼란을 줄 수 있다.  
이처럼 타입 변환은 객체 전체의 변환 또는 연산을 수행하는 것이므로 메서드 또는 확장 함수를 사용하는 것이 더 적절하다.

**6. 'getter'가 프로퍼티 상태를 변경하는 경우** :  
프로퍼티의 'getter'는 프로퍼티의 현재 값을 조회하는 용도로만 사용되어야 하며, 상태를 변경하거나 다른 Side-effect를 발생시켜서는 안된다.  
이러한 접근 방식은 코드 가독성과 안정성, 캡슐화를 해칠 수 있게되며, 프로퍼티의 상태 변경이 필요한 경우에는 명시적으로 메서드를 사용하는 것이 더 적절하다.

예를 들어, 요소들의 합을 계산하는 것은 모든 요소를 순회해야 하며, 이는 선형 복잡도를 가지게 된다.  
따라서 이는 프로퍼티가 아니여야 하며, 실제로 표준 라이브러리에서도 함수로 정의되어야 한다.

```kotlin
// val s = (1..100).sum // No, it would be a violation of this rule
val s = (1..100).sum()
```

상태를 나타내고 설정하기 위해 프로퍼티를 사용하는데, 특별한 이유가 없다면 아래와 같이 'setter / getter'를 사용하는것이 좋다. 

```kotlin
// Don't do this
class UserIncorrect {
    private var name: String = ""
  
    fun getName() = name
    
    fun setName(name: String) {
        this.name = name
    }
}

// Do this
class UserCorrect {
    var name: String = ""
}
```