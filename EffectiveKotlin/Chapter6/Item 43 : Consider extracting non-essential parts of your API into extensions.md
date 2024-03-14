클래스 내에서 'final' 메서드를 구현할 때, 해당 메서드를 클래스의 멤버로 포함시킬지, 아니면 확장 함수로 정의할 지 결정해야 한다.

```kotlin
// Defining methods as members
class Workshop(/* ... */) {
    // ...
    
    fun makeEvent(date: DateTime): Event = // ...
    
    val permalink 
        get() = "/workshop/$name"
}

// Defining methods as extensions
class Workshop(/* ... */) {
    // ...
}

fun Workshop.makeEvent(date: DateTime): Event = // ...

val Workshop.permalink
    get() = "/workshop/$name" 
```

이 두 방식은 여러 가지로 서로 유사하다.  
이들을 사용하는 방법이나 리플렉션을 이용해 참조하는 방법 역시 매우 유사하다.

```kotlin
fun useWorkshop(workshop: Workshop) {
    val event = workshop.makeEvent(date)
    val permalink = workshop.permalink
    
    val makeEventRef = Workshop::makeEvent
    val permalinkPropRef = Workshop::permalink
}
```

그럼에도 불구하고, 두 방식 사이에는 명확한 차이점이 존재한다.  
각각의 방식은 장점과 단점을 지니고 있으며, 한 방식이 다른 방식을 압도하지 않는다.  
이러한 이유로, 반드시 모든 부분을 확장으로 추출하는 것이 아니라, 이런 추출을 고려해 볼 것을 제안하는 것이다.  
중요한 것은 이러한 차이점을 정확히 이해하고 올바른 결정을 내리는 것이다.

멤버와 확장 사이의 가장 큰 차이는 사용법에 있어서 확장은 별도로 'import' 한다는 점이다. 따라서 확장은 다른 패키지에 배치될 수 있는 유연성을 가진다.   
이러한 특성은 직접 클래스에 멤버를 추가할 수 없는 상황이나, 데이터와 행동을 분리하고자 하는 프로젝트 구조에서 유용하게 사용될 수 있다.  
클래스 내부에는 필드를 포함하는 프로퍼티가 필요하지만, 메서드는 클래스의 'public api'만을 사용한다면, 이를 클래스 외부에 위치시킬 수 있는 가능성을 열어 준다.

확장 함수가 별도로 'import' 되어야 한다는 점 덕분에, 같은 타입에 대하여 동일한 이름을 가진 여러 확장을 가질 수 있다.  
이는 여러 라이브러리에서 추가적인 메서드를 제공할 수 있도록 하며, 이로 인해 함수 이름에 대한 충돌의 우려가 줄어든다.  
그러나, 이름은 같지만 동작이 다른 두 확장 함수가 존재한다는 점은 위험할 수 있다. 이런 상황에서는 멤버 함수를 생성함으로써 문제를 해결할 수 있다.  
컴파일러는 확장 함수보다는 항상 멤버 함수를 선택하는 경향이 있으므로, 이를 통해 충돌 문제를 명확하게 해결 할 수 있다. (고르디우스의 매듭을 잘라 낼 수 있다.)

확장 함수는 '가상'이 아니라는 점에서 중요한 차이점을 가진다. 즉, 파생 클래스에서 재정의 될 수 없는 중요한 특징을 가진다.  
호출할 확장 함수는 컴파일 중에 정적으로 선택된다. 이는 Kotlin에서 멤버 요소가 가상인 것과는 다른 동작이다.   
따라서, 상속을 위해 설계된 요소에 대해서는 확장을 사용하는 것은 적절하지 않다. 

```kotlin
open class C
class D: C()

fun C.foo() = "c"
fun D.foo() = "d"

fun main() {
    val d = D()
    println(d.foo())            // d
    val c: C = d
    println(c.foo())            // c 
    
    println(D().foo())          // d
    println((D() as C).foo())   // c
}
```

위와 같은 동작은 확장 함수가 실제로 컴파일 단계에서, 확장 함수의 수신 객체를 첫 번째 인자로 받는 일반 함수로 변환하기 때문에 가능하다.

```kotlin
fun foo(`this$receiver`: C) = "c"
fun foo(`this$receiver`: D) = "d"

fun main() {
    val d = D()
    println(foo(d))             // d
    val c: C = d
    println(foo(c))             // c 
    
    println(foo(D()))           // d
    println(foo(D() as C))      // c
}
```

위와 같은 원리로 인해 발생하는 또 다른 중요한 특징은 확장을 클래스가 아닌 타입에 정의한다는 점이다.   
이는 개발자에게 더 큰 유연성을 제공한다.

예를 들어, 'nullable' 타입이나 제네릭 타입의 특정 구현에 확장 함수를 정의하는 것이 가능하다.

```kotlin
inline fun CharSequence?.isNullOrBlank(): Boolean {
    contract { 
        returns(false) implies (this@isNullOrBlank != null) 
    }
    
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

마지막으로 주목해야 할 중요한 차이는 확장이 클래스 참조에 멤버로 표시되지 않는다는 점이다.  
이로 인해 'annotation processor' 중에는 확장 함수를 고려 대상에 포함시키지 않게 된다.  
따라서 'annotation processing' 과정에서 클래스의 일부로 처리해야 할 요소들을 확장 함수로 변환하는 것은 불가능하다.  

반면에, 중요하지 않은 요소들을 확장 함수로 분리함으로써, 이러한 'processor' 들에 의해 인식될 우려 없이 코드를 정리할 수 있다.  
이러한 요소들은 원래 클래스 내에 포함되지 않기에 별도로 숨길 필요가 없다.