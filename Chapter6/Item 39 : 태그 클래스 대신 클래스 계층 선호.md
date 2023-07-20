# Item 39: 태그 클래스 보다는 클래스 계층을 선호

규모가 큰 프로젝트에서는 "모드" 라는 상수가 있는 클래스를 종종 볼 수 있습니다. 
이 "모드"는 클래스가 어떻게 동작해야 하는지를 지정하며 이러한 클래스를 태그 클래스 라고 부릅니다.

태그 클래스는 다양한 문제점을 가지고 있습니다.
대부분의 문제는 "서로 다른 모드"에서 나오는 "다양한 책임들이 같은 클래스 내부"에 있기에 발생됩니다.

아래는 테스트에서 어떤 값이 특정 기준을 충족하는지 확인하는데 사용되는 클래스의 예시입니다.

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
    
    enum class Matcher { EQUAL, NOT_EQUAL, LIST_EMPTY, LIST_NOT_EMPTY }
    
    companion object {
        fun <T> equal(value: T) = ValueMatcher(value = value, matcher = Matcher.EQUAL)
        fun <T> notEqual(value: T) = ValueMatcher(value = value, matcher = Matcher.NOT_EQUAL)
        fun <T> listEmpty() = ValueMatcher<T>(matcher = Matcher.LIST_EMPTY)
        fun <T> listNotEmpty() = ValueMatcher<T>(matcher = Matcher.LIST_NOT_EMPTY)
    }
}
```

이러한 방식의 접근은 여러 단점이 있습니다.
1. 하나의 클래스에서 여러 모드를 처리하기 위한 추가적인 보일러플레이트 필요
2. 일관성 없이 사용되는 프로퍼티들이 있으며 이들은 다양한 목적으로 사용되며, 객체가 일반적으로 필요한 것보다 더 많은 프로퍼티를 가질 수 있음 (예시에서는 `value`가 `LIST_EMPTY`나 `LIST_NOT_EMPTY` 모드에서는 사용되지 않음)
3. 요소들이 다목적으로 사용되며, 여러 방법으로 설정될 수 있을 때 상태의 일관성과 정확성 보호가 어려움
4. 객체가 올바르게 생성되도록 보장하기 위해 팩토리 메서드를 사용해야 함

이와 같은 태그 클래스 대신, Kotlin에서는 `sealed class`를 사용하도록 권장합니다.

하나의 클래스에 여러 모드를 축적하는 대신 각 모드에 대한 클래스를 정의하고 이들의 다형성을 허용하는 방식으로 사용될 수 있도록 타입 시스템을 사용하는 것이 좋습니다.
추가적으로 `sealed` Modifier 추가하여 이러한 클래스들을 하나의 집합으로 제한합니다.

```kotlin
sealed class ValueMatcher<T> {
    
    abstract fun match(value: T): Boolean 
    
    class Equal<T>(val value: T): ValueMatcher<T>() {
        override fun match(value: T) = this.value == value
    }
    
    class NotEqual<T>(val value: T): ValueMatcher<T>() {
        override fun match(value: T) = this.value != value
    }
    
    class ListEmpty<T>: ValueMatcher<List<T>>() {
        override fun match(value: List<T>) = value is List<*> && value.isEmpty()
    }
    
    class ListNotEmpty<T>: ValueMatcher<List<T>>() {
        override fun match(value: List<T>) = value is List<*> && value.isNotEmpty()
    }
}
```

이러한 구현은 여러 책임이 서로 얽히지 않아 훨씬 깔끔하며, 각 객체는 필요한 데이터만 가지고 있으며 어떤 파라미터를 받을지 정의할 수 있습니다.
이처럼 타입 계층 구조를 사용하면 태그 클래스의 모든 단점을 제거할 수 있습니다.

---

## Sealed modifier

Kotlin의 sealed modifier는 유용하며, `sealed` 클래스는 모든 하위 클래스 한정으로 동일한 파일에 있도록 제한할 수 있습니다.
이러한 제한은 `when`문을 통해 모든 경우의 수를 처리하는 것이 보장됩니다.
이는 `when`문에서 `else` 구문을 사용할 필요가 없다는 것을 의미합니다.

이러한 이점을 활용하면 새로운 기능을 간편하게 추가할 수 있고 `when`문에서 빠트린 처리가 없도록 확인할 수 있습니다.

이와 다르게, `abstract` 클래스를 사용하면 다른 개발자가 새로운 하위 클래스를 만들 수 있기에 새로운 하위 클래스에 대해 `when` 표현식이 처리하지 못하는 경우가 발생할 수 있습니다.
이 때문에 `abstract` 클래스에서는 `when` 표현식 대신 함수를 `abstract`로 선언하고 각 하위 클래에서 해당 함수를 오버라이드 하는 것이 좋습니다.

아래 방식은 `sealed` 클래스를 사용함으로써 `when` 표현식을 안전하게 사용할 수 있음을 보여줍니다.
또한 새로운 함수를 확장하기 간편하며 새로운 기능을 추가하기도 쉽습니다. 
만약 `ValueMatcher`가 `abstract` 클래스 였다면, 새로운 기능을 추가하는것이 까다롭고 최악의 경우 불가능할 수 있습니다.

```kotlin
fun <T> ValueMatcher<T>.reversed(): ValueMatcher<T> = when(this) {
    is ValueMatcher.Equal -> ValueMatcher.NotEqual(value)
    is ValueMatcher.NotEqual -> ValueMatcher.Equal(value)
    is ValueMatcher.ListEmpty -> ValueMatcher.ListNotEmpty()
    is ValueMatcher.ListNotEmpty -> ValueMatcher.ListEmpty()
}
```

이와 같이 무조건 `sealed`를 사용하는 것이 아닌 때에 따라 `abstract`를 사용하는 경우도 생각할 수 있습니다.

---

## 태그 클래스 != 상태 패턴

태그 클래스와 상태 패턴은 동일한 것이 아닙니다.

상태 패턴은 객체 내부 상태가 변경될 때 그 행동을 변경하게 하는 행동적인 소프트웨어 디자인 패턴입니다.
주로 MVC, MVP, MVVM 등의 아키텍쳐에 따라 Controller, Presenter, ViewModel 등에 사용됩니다.

또한 상태 패턴 사용 시 다양한 상태를 나타내는 클래스 계층과 현재 상태를 나타내는 변경 가능한 프로퍼티를 가지게 됩니다.

예를 들어 아침 운동을 위한 앱을 작성하는 경우 각 운동 시작 전에 준비 시간이 있고, 운동이 끝나면 운동이 끝났다는 화면이 나타납니다.

```kotlin
sealed class WorkoutState

class PreParerState(val exercies: Exercise): WorkoutState()
class ExerciseState(val exercise: Exercise): WorkoutState()
object DoneState : WorkoutState()

fun List<Exercise>.toState(): List<WorkoutState> = 
    flatMap { exercise ->
        listOf(PreParerState(exercise), ExerciseState(exercise))
    } + DoneState

class WorkoutPresenter(/* ... */) {
    private var state: WorkoutState = states.first()
    
    // ...
}
```

다음과 같은 차이점이 있습니다.
- 상태는 더 많은 책임을 갖는 더 큰 클래스의 일부 입니다.
- 상태는 변화가 필요합니다.

상태는 대부분 변경 가능한 단일 프로퍼티인 `State`에서 유지됩니다.

구체적인 상태는 객체로 표현되며, 객체는 태그 클래스가 아닌 `sealed` 클래스 계층을 사용하는 것을 권장합니다.
또한, 객체는 불변성을 유지하며, 상태를 변경해야 할 떄는 `state` 프로퍼티를 변경해야 합니다.

상태가 변경될 때 마다 뷰를 갱시하기 위해 상태가 관찰되는 경우도 흔합니다.

```kotlin
private var state: WorkoutState by Delegates.observable(states.first()) { _, _, _ -> 
    updateView()
}
```
