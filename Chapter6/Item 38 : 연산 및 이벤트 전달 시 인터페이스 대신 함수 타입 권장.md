# Item 38: 연산 및 이벤트 전달 시 인터페이스 대신 함수 타입을 사용할 것

많은 언어들이 함수 타입의 개념을 가지진 않지만 단일 메서드를 가진 인터페이스를 사용하고 있습니다.

이러한 인터페이스들은 SAM(Single Abstract Method)라고 불립니다.

아래는 뷰 클릭 이벤트 시 어떤 일이 일어나는 지에 대한 정보를 전달하는 SAM의 예시입니다.

함수가 SAM를 기대할 때, 이 인터페이스를 구현하는 `object` 인스턴스를 전달해야 합니다.

```kotlin
interface OnClick {
    fun onClick(view: View)
}

fun setOnClick(onClick: OnClick) { /* ... */ }

setOnClick(object : OnClick {
    override fun onClick(view: View) {
        TODO("Not yet implemented")
    }
})
```

그러나 파라미터를 함수 타입으로 선언할 경우 더 많은 자유도를 얻을 수 있습니다. 

```kotlin
fun setOnClick(onClick: (View) -> Unit) { /* ... */ }
```

아래와 같이 파라미터를 다음과 같이 전달할 수 있습니다.

- 람다 표현식 또는 익명 함수

```kotlin
setOnClick { view -> /* ... */ }
setOnClick(fun(view: View) { /* ... */ })
```

• 함수 참조 또는 바운드 함수 참조

```kotlin
setOnClick(::handleClick)
setOnClick(this::showUsers)
```

• 선언된 함수 타입을 구현하는 객체

```kotlin
class ClickListener : (View) -> Unit {
    override fun invoke(view: View) { /* ... */ }
}

setOnClick(ClickListener())
```

위 옵션들은 더 넓은 범위의 사용 사례를 다룰 수 있습니다.

반면에 SAM의 장점은 함수의 파라미터 값과 반환 값이 명확하다는것 입니다.

함수 타입에서는 이러한 점을 보완하고자 `typealias`을 제공하고 있습니다.

`typealias`는 함수 타입에 이름을 지정할 수 있어, 함수의 파라미터와 반환 값에 대한 정보를 명확하게 나타낼 수 있습니다.
이는 함수 타입의 유연성을 유지하면서 코드의 명시성을 높입니다.

```kotlin
typealias OnClick = (View) -> Unit
```

파라미터에도 `typealias`를 붙여 이들이 IDE에 의해 기본적으로 제안 될 수 있도록 할 수 있습니다.

```kotlin
fun setOnClick(listener: OnClick) { /* ... */ }
typealias OnClick = (view: View) -> Unit
```

람다 표현식을 사용할 때, 개발자들은 인수를 분해(destructure)할 수 있습니다.

이 모든 것이 종합되어 함수 타입은 일반적으로 SAM 보다 좋은 옵션이 될 수 있습니다.

이러한 주장을 뒷받침하는 것중 하나는 많은 observer를 설정할 때 좋은 사례가 될 수 있습니다.
클래식한 Java 방식은 종종 이들을 단일 리스너 인터페이스에 모읍니다.

```kotlin
class CalendarView {
    var listener: Listener? = null
    
    interface Listener {
        fun onDateSelected(date: LocalDate)
        fun onPageChanged(date: LocalDate)
    }
}
```

API 사용자 입장에서는 위 리스너들을 함수 타입을 갖는 별개의 프로퍼티로 설정하는 것이 좋을 수 있습니다.

```kotlin
class CalendarView {
    var onDateSelected: ((LocalDate) -> Unit)? = null
    var onPageChanged: ((LocalDate) -> Unit)? = null
}
```

위와 같이 변경하면, `onDateSelected`와 `onPageChanged`의 구현이 인터페이스에서 묶여야 할 필요가 없으며 이 함수들을 독립적으로 변경 될 수 있습니다.

이처럼 인터페이스를 정의할 타당한 이유가 없다면, 함수 타입을 사용하는것이 더 좋습니다.