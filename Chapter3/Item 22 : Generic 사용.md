# Item 22 : 공통 알고리즘 구현 시 제네릭을 사용

함수에 인자(Argument)를 전달하는 것처럼, 타입(Type)을 타입 인자(Type Arguemnt)로 전달할 수 있습니다.  
타입 인자를 받는 함수 (='타입 매개 변수'를 가지는 함수)를 제네릭(generic) 함수라고 합니다.

아래의 `filter`는 타입 매개변수 `T`를 가지는 예시 입니다.

```kotlin
inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    val destination = ArrayList<T>()
    for (element in this) {
        if (predicate(element)) destination.add(element)
    }
    
    return destination
}
```

컴파일러는 타입 매개변수를 통해 타입을 좀 더 정확하게 확인하고 추론할 수 있어 프로그램의 안전성을 향상시킬 수 있습니다.

> reified type을 제외한 제네릭은 일반적으로 JVM 타입 소거 특성으로 인해 타입 정보는 대부분 컴파일 시간에만 유용하고 런타임에는 제거 됩니다.

제네릭은 주로 클래스와 인터페이스에 도입되어, 구체적인 타입만을 가지는 컬렉션의 생성을 가능하게 합니다. 
예를 들어, `MutableList<Int>`는 `Int` 타입의 요소만을 가질 수 있도록 강제합니다. 
이런 방식으로, 컴파일러는 우리가 적절한 타입의 요소만을 컬렉션에 추가하도록 강제하고, 컬렉션에서 요소를 가져올 때는 올바른 타입을 반환하게 합니다.
