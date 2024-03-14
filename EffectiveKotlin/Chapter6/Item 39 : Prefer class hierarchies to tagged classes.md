큰 프로젝트를 진행하다 보면, 클래스가 어떻게 동작하는지를 지정하는 'mode'라는 'constant'를 포함한 클래스들을 볼 수 있다.  
이런 클래스들을 '**태그가 붙은 클래스**'라고 부르며, 이 클래스들은 '동작 모드'를 나타내는 '태그'로 인해 여러 문제가 발생한다.

대부분의 문제는 서로 다른 '모드'에서 나오는 다양한 책임들이 하나의 클래스 안에서 공간을 차지하기 위해 경쟁하고 있기 때문에 발생한다.  
이 책임들은 일반적으로 서로 구분 될 수 있다. 

예를 들어, 아래 코드는 단순화 되었지만, 어떤 값이 특정 기준을 만족하는지 테스트하기 위해 사용되는 클래스이다.

```kotlin
class ValueMatcher<T> private constructor(
    private val value: T? = null,
    private val matcher: Matcher
) {
    fun match(value: T?) = when(matcher) {
        Matcher.EQUAL -> value == this.value
        Matcher.NOT_EQUAL -> value != this.value
        Matcher.LIST_EMPTY -> value is List<*> && value.isEmpty()
        Matcher.LIST_NOT_EMPTY -> value is List<*> && value.isNotEmpty()
    }
    
    enum class Matcher { 
        EQUAL,
        NOT_EQUAL,
        LIST_EMPTY,
        LIST_NOT_EMPTY 
    }
    
    companion object {
        fun <T> equal(value: T) = ValueMatcher(value = value, matcher = Matcher.EQUAL)
        fun <T> notEqual(value: T) = ValueMatcher(value = value, matcher = Matcher.NOT_EQUAL)
        fun <T> listEmpty() = ValueMatcher<T>(matcher = Matcher.LIST_EMPTY)
        fun <T> listNotEmpty() = ValueMatcher<T>(matcher = Matcher.LIST_NOT_EMPTY)
    }
}
```

이 접근 방식에는 여러 가지 단점이 있다.

- 하나의 클래스 안에서 여러 모드를 관리하면서 생기는 추가적인 보일러플레이트
- 다양한 목적으로 사용되는 프로퍼티들로 인해 발생하는 부족한 일관성. 
  - 이로 인해, 객체는 다른 모드에 필요 할 수 있는 여분의 프로퍼티들로 인해 필요 이상의 프로퍼티를 가질 수 있다.
  - 예를 들어, 위 예에서 'LIST_EMPTY' 또는 'LIST_NOT_EMPTY' 모드일 때 'value'가 사용되지 않는다.
- 요소들이 여러 목적으로 사용될 수 있고, 다양한 방법으로 설정될 수 있기에, 상태의 일관성과 정확성을 유지하기가 어렵다.
- 객체가 정확하게 생성되도록 보장하는 것이 어렵기에, 대부분의 경우 팩토리 함수를 사용해야 한다.

Kotlin에서는 '태그가 붙은 클래스'를 사용하는 대신, 더 좋은 대안인 'sealed class'가 있다.  
하나의 클래스 안에 여러 모드를 모으는 것이 아니라, 각 모드마다 별도의 클래스를 정의하고 이들을 다형적으로 사용할 수 있게 하는 것이 좋다.  
이럴 때 'sealed' modifier를 사용하면, 이 클래스들을 'set of alternatives'으로 '봉인' 할 수 있다.

아래는 위 예시를 'sealed class'를 사용하여 구현한 것이다.

```kotlin
sealed class ValueMatcher<T> {
    
    abstract fun match(value: T): Boolean 
    
    class Equal<T>(val value: T): ValueMatcher<T>() {
        override fun match(value: T) = 
            this.value == value
    }
    
    class NotEqual<T>(val value: T): ValueMatcher<T>() {
        override fun match(value: T) = 
            this.value != value
    }
    
    class ListEmpty<T>: ValueMatcher<List<T>>() {
        override fun match(value: List<T>) = 
            value is List<*> && value.isEmpty()
    }
    
    class ListNotEmpty<T>: ValueMatcher<List<T>>() {
        override fun match(value: List<T>) = 
            value is List<*> && value.isNotEmpty()
    }
}
```

위와 같은 구현은 서로 다른 책임들이 복잡하게 얽히지 않아 훨씬 더 깔끔하다.  
또한, 각 객체는 자신에게 필요한 데이터만 보유하고, 자신이 필요로 하는 파라미터를 스스로 정의할 수 있다.  

결과적으로 타입 계층 구조를 사용함으로써, '태그가 붙은 클래스'들의 모든 문제점들을 해결할 수 있다.

---

## Sealed modifier

'sealed' modifier를 필수적으로 사용할 필요는 없다. 대신에 'abstract'를 사용할 수 있지만, 'sealed'는 해당 파일 외부에서 하위 클래스를 정의하는 것을 금지해주는 특징이 있다.
덕분에, 'when' 문에서 모든 경우를 다룬다면, 모든 경우가 이미 고려되었기 때문에 'else' 분기를 추가할 필요가 없음을 보장한다.  
이런 장점을 통해, 새로운 기능을 쉽게 추가할 수 있고, 이 'when' 문에서 해당 기능들을 다루는 것을 잊어버리는 일이 없도록 할 수 있다.

이 방법은 다른 모드에 따라 다르게 동작하는 연산들을 정의하는 데 있어 매우 편리하다.  
예를 들어, 각각의 하위 클래스마다 동작 방식을 별도로 정의하는 것이 아니라, 'when'문을 통해 'match' 함수를 정의할 수 있다.  
이 방법으로 새로운 함수들을 확장 함수 형태로도 추가할 수 있다.

```kotlin
fun <T> ValueMatcher<T>.reversed(): ValueMatcher<T> = 
    when(this) {
        is ValueMatcher.Equal -> ValueMatcher.NotEqual(value)
        is ValueMatcher.NotEqual -> ValueMatcher.Equal(value)
        is ValueMatcher.ListEmpty -> ValueMatcher.ListNotEmpty()
        is ValueMatcher.ListNotEmpty -> ValueMatcher.ListEmpty()
    }
```

다른 한편으로, 'abstract'를 사용하면, 관리자의 범위 밖에서 다른 개발자들이 새로운 인스턴스를 만들 수 있는 가능성을 열어준다.  
이러한 경우, 관리자는 함수를 'abstract'로 선언하고, 이 함수들을 각각의 하위 클래스에서 구현해야 한다.  
왜냐하면 'when'문을 사용할 때, 프로젝트의 범위를 벗어나 새로운 클래스가 추가되면, 해당 함수가 제대로 동작하지 않을 위험이 있기 때문이다.

---

## Tagged classes are not the same as state pattern

'태그가 붙은 클래스'와 '상태 패턴'을 혼동해서는 안된다.  
'상태 패턴'은 객체가 내부 상태의 변화에 따라 그 행동을 바꿀 수 있게하는 소프트웨어 디자인 패턴이다.  
이 패턴은 주로 프론트엔드의 'Controller', 'Presenter', 'ViewModel'(MVC, MVP, MVVM 아키텍처에 해당)에서 사용된다.

예를 들어, 아침 운동 앱을 가정해보면, 각 운동 전에는 준비 시간이 있고, 마지막에는 운동을 마쳤다는 화면이 나타날 것이다.

'상태 패턴'을 사용하면, 서로 다른 상태들을 대표하는 클래스들의 계층 구조를 구성하게 된다.  
또한, 현재 진행 중인 상태가 무엇인지를 나타내는데 사용되는 'read-write 프로퍼티'가 있다.

```kotlin
sealed class WorkoutState { 
    class PreParerState(val exercies: Exercise): WorkoutState()
    class ExerciseState(val exercise: Exercise): WorkoutState()
    data object DoneState : WorkoutState()
}

fun List<Exercise>.toState(): List<WorkoutState> = 
    flatMap { exercise ->
        listOf(
          PreParerState(exercise),
          ExerciseState(exercise)
        )
    } + DoneState

class WorkoutPresenter(/* ... */) {
    
    private var state: WorkoutState = states.first()
    // ...
}
```

'상태 패턴'과 '태그가 붙은 클래스'의 주된 차이점은 다음과 같다.

- 상태는 보다 많은 책임을 지닌 더 큰 클래스의 일부분이다.
- 상태는 변경되어야 한다.

상태는 보통 'state'라는 'read-write 프로퍼티'에 저장된다.  
구체적인 상태는 객체를 통해 나타내며, 이 때 '태그가 붙은 클래스' 보다는 'sealed class'의 계층 구조를 사용하는 것을 선호한다.  
이 객체를 불변 객체로 유지하며, 상태를 변경할 필요가 있을 때는 'state' 프로퍼티 자체를 변경한다.  
또한, 아래와 같이 상태가 바뀔 때 마다 'View'를 갱신하기 위해 상태를 관찰하는 것은 일반적인 일이다.

```kotlin
private var state: WorkoutState by Delegates.observable(states.first()) { _, _, _ -> 
    updateView()
}
```
