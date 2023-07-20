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
