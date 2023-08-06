# Item 43: API 비핵심 부분을 extension으로 추출하는 것을 고려

최종적으로 클래스에서 메서드 정의 시, 메서드를 멤버로 정의할 것인지 확장 함수로 정의할 것인지 결정해야 합니다.

```kotlin
// 메서드를 멤버로 정의
class Workshop() {
    val permalink 
        get() = "/workshop/$name" 
    
    fun makeEvent(date: DateTime): Event = // ...
}

// 메서드를 확장으로 정의
class Workshop() {
    // ...
}

fun Workshop.makeEvent(date: DateTime): Event = // ...

val Workshop.permalink
    get() = "/workshop/$name" 
```

두 접근법은 사용 방법 및 리플렉션을 통한 참조도 매우 유사합니다. 

```kotlin
fun useWorkshop(workshop: Workshop) {
    val event = workshop.makeEvent(date)
    val permalink = workshop.permalink
    
    val makeEventRef = Workshop::makeEvent
    val permalinkRef = Workshop::permalink
}
```

두 접근법에는 각각 장단점이 존재하며 한 가지 방식이 다른 방식을 우위에 두고 있지 않습니다.
이 때문에 메서드를 확장(Extension)으로 반드시 추출하는 것이 아니라 두 접근법의 차이점을 이해하고 올바른 결정을 해야 합니다.

두 접근법의 가장 큰 차이점 중 하나는 Extension의 경우 별도로 임포트해야 한다는 것입니다.
이로인해 Extension은 다른 패키지에 위치할 수 있으며 이는 멤버를 직접 추가할 수 없는 경우에 유용하게 사용될 수 있습니다.
또한 데이터와 동작을 분리하도록 설계된 프로젝트에서도 Extension은 좋은 방식으로 사용될 수 있습니다.

즉, 필드가 있는 프로퍼티는 클래스 내부에 위치해야 하지만, 클래스 공용 API에만 접근하는 메서드는 별도로 위치할 수 있습니다.

Extension을 별도로 임포트해야 한다는 점 덕분에 동일한 유형에 동일한 이름을 가진 많은 Extension을 가질 수 있습니다.
이는 다른 라이브러리가 추가적인 메서드를 제공할 수 있기에 좋은 방식입니다.

그러나 동일한 이름을 가지며 다른 동작을 가진 두 Extension이 있는 경우 위험할 수 있습니다.
이러한 경우 멤버 함수를 만들어 고르디우스의 매듭을 잘라낼 수 있습니다. 또한 컴파일러는 항상 Extension 보다 멤버 함수를 우선 선택 합니다.

또 다른 주요 차이점은 Extension은 가상이 아니라는 것입니다. 즉, 파생 클래스에서 재정의할 수 없으며 확장(Extension) 함수는 컴파일 중에 정적으로 선택됩니다.

이는 Kotlin에서 멤버 요소가 가상인 것과 다른 동작입니다. 따라서 상속을 위해 설계된 요소에 대해선 Extension을 사용하지 않아야 합니다.

```kotlin
open class C
class D: C()

fun C.foo() = "c"
fun D.foo() = "d"

fun main() {
    val d = D()
    println(d.foo()) // d
    val c: C = d
    println(c.foo()) // c 
    
    println(D().foo()) // d
    println((D() as C).foo()) // c
}
```

위 동작은 확장 함수가 첫 번째 인수로 확장 수신자를 두는 일반 함수로 컴파일 되는 결과 입니다.

```kotlin
fun foo(`this$receiver`: C) = "c"
fun foo(`this$receiver`: D) = "d"

fun main() {
    val d = D()
    println(foo(d)) // d
    val c: C = d
    println(foo(c)) // c 
    
    println(foo(D())) // d
    println(foo(D() as C)) // c
}
```

위 코드를 통해 개발자들은 확장을 클래스가 아니라 타입에 정의할 수 있음을 알 수 있습니다. 이러한 방식은 더 많은 유연성을 제공합니다.  
예를 들어 확장을 `nullable` 또는 generic type으로 정의할 수 있습니다.

```kotlin
inline fun CharSequence?.isNullOrBlank(): Boolean {
    contract { returns(false) implies (this@isNullOrBlank != null) }
    
    return this == null || this.isBlank()
}

public fun Iterable<Int>.sum(): Int {
    var sum: Int = 0
    for (element in this) {
        sum += element
    }
    
    return sum
}
```

마지막으로 두 접근법의 중요한 차이점은 확장이 클래스 참조에 있어 멤버로 나열되지 않는다는 것입니다.
확장은 실제로는 클래스의 인스턴스에 대한 첫 번째 파라미터를 가진 정적 메서드로 컴파일되기 때문에 클래스 참조를 통해 액세스할 수 없습니다.

이로 인해, 어노테이션 프로세서와 같은 도구는 확장 함수나 프로퍼티를 클래스의 멤버로 간주하지 않습니다.
어노테이션 프로세서는 주로 컴파일 시 어노테이션이 붙은 코드를 분석하기 처리하기 위해 사용됩니다.
이 때, 확장 함수나 프로퍼티는 어노테이션 프로세서의 대상이 아닙니다.
따라서 클래스를 어노테이션을 사용하여 처리할 때, 처리해야 하는 요소를 확장 함수로 추출할 수 없습니다.

반면에, 비핵심 요소를 확장으로 추출한다면 이들이 어노테이션 프로세서에 의해 확인되는 것에 대해 걱정할 필요가 없습니다.
비핵심 요소를 클래스 외부로 이동시키면, 어노테이션 프로세서와 같은 도구는 이 요소들을 확인할 수 없습니다.
따라서 이러한 요소들은 클래스 내에 있지 않기에 숨길 필요가 없습니다.
이는 코드의 복잡성을 줄이고, 프로세서가 실제로 처리해야 하는 클래스의 핵심 요소에만 집중할 수 있게 합니다. 
