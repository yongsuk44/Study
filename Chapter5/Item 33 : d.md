# Item 33: Consider factory functions instead of constructors

Kotlin에서 클래스의 인스턴스를 얻는 가장 일반적인 방법은 기본 생성자(constructor)를 통해 얻는 방법입니다.

```kotlin
class MyLinkedList<T> constructor(val head: T, val tail: MyLinkedList<T>?)

val list = MyLinkedList(1, MyLinkedList(2, null))
```

생성자를 통해 객체의 생성하는 다른 방법이 있습니다.
객체 인스턴스화를 위한 다양한 패턴들이 있는데, 대부분은 직접 객체를 생성하는 대신 함수가 이 역할을 대신 해주는 아이디어를 중심으로 돌아갑니다.

아래는 최상위 함수 `MyLinkedList`의 인스턴스를 생성하는 예시 입니다.

```kotlin
fun <T> myLinkedListOf(vararg element: T): MyLinkedList<T>? {
    if(element.isEmpty()) return null
    
    val head = element.first()
    val elementTail = element.copyOfRange(1, element.size)
    val tail = myLinkedListOf(*elementTail)
    return MyLinkedList(head, tail)
}

val list = myLinkedListOf(1, 2)
```

위와 같이 생성사 대신 사용되는 함수를 팩토리(Factory) 함수라 하고, 이 함수를 사용하면 다음과 같은 장점들이 있습니다.

- 생성자와 달리 함수는 이름이 존재합니다. 함수의 이름은 객체가 어떻게 생성되고 인수가 무엇인지를 설명할 수 있습니다. 예를 들어, `ArrayList(3)`와 같이 인수가 어떤 의미지 명확하게 이해할 수 없습니다. 이러한 경우에는 `ArrayList.withSize(3)`와 같이 이름에 혼동을 방지해줄 수 있습니다.
- 함수는 생성자와 달리 반환 유형의 모든 하위 유형의 객체를 반환할 수 있습니다. 이는 각기 다른 객체를 사용해야 하는 경우에 더 나은 객체를 제공하는 데 사용할 수 있습니다.
- 함수는 호출할 때마다 새로운 객체를 생성할 필요가 없습니다. 함수를 사용하여 객체를 생성하면 캐싱 메커니즘을 포함시켜 객체 생성을 최적화하거나 특정 경우에 객체를 재사용할 수 있습니다.
- 객체 외부에 팩토리 함수 정의 시 가시성을 제어하여 같은 파일 이나 같은 모듈에서만 접근할 수 있도록 최상위 팩토리 함수를 생성할 수 있습니다.
- 팩토리 함수는 `inline`와 함께 사용할 경우 타입 파라미터를 실체화(`reified`) 할 수 있습니다.
- 팩토리 함수는 복잡하게 생성될 수 있는 객체를 생성할 수 있습니다.
- 생성자는 상위 클래스의 생성자나 기본 생성자를 즉시 호출해야 하지만, 팩토리 함수 사용 시 생성자 사용을 연기할 수 있습니다.

팩토리 함수는 위와 같은 장점들이 있지만, 기본 생성자와 경쟁하는 것이 아니라는 것을 이해해야 합니다. 
팩토리 함수는 본문에서 생성자를 사용해야 하므로 생성자는 반드시 존재해야 합니다.

Kotlin에서는 다양한 종류의 팩토리 함수가 존재합니다. 
- Compoinion object factory function
- Extension factory function
- Top-level factory functions
- Fake constructors
- Methods on a factory class