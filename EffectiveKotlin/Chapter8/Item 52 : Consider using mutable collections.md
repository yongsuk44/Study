가변 컬렉션을 사용하는 가장 큰 장점은 성능 측면에서 더 빠르다는 것이다.  
불변 컬렉션에 요소를 추가하려면, 우선 새로운 컬렉션을 만들고, 그 안에 기존의 모든 요소들을 다시 추가해야만 한다.  
아래는 현재 Kotlin 표준 라이브러리에서 구현된 방식이다.

```kotlin
public operator fun <T> Iterable<T>.plus(element: T): List<T> {
    if (this is Collection) return this.plus(element)
    
    val result = ArrayList<T>()
    result.addAll(this)
    result.add(element)
    return result
}
```

위와 같이 이전 컬렉션에서 모든 요소를 가져오는 작업은 큰 컬렉션을 처리할 때 상당한 비용이 발생한다.  
이는 요소를 추가해야 하는 상황에서 가변 컬렉션을 사용하는 것이 성능을 향상시키는 핵심 방법임을 의미한다.  

그러나 'Item 1'에서는 불변 컬렉션 사용이 안전성 면에서 제공하는 장점을 강조한다.  
하지만, 이러한 주장은 동기화나 캡슐화가 거의 필요하지 않은 지역 변수의 경우에는 드물게 적용됨을 알아야 한다.  
따라서 지역적인 데이터 처리에 있어 가변 컬렉션을 사용하는 것이 대체로 더 적합하다.  
이는 모든 컬렉션 처리 기능이 내부적으로 가변 컬렉션을 이용해 구현된 표준 라이브러리에서도 확인할 수 있는 사실이다.

```kotlin
inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R> {
    val destination = ArrayList<R>(
        if (this is Collection<*>) this.size 
        else 10
    )
    
    for (item in this)  {
        destination.add(transform(item))
    }
    
    return destination
}
```

불변 컬렉션을 사용하는 대신 가변 컬렉션의 사용

```kotlin
// This is not how map is implemented
inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R> {
    val destination = listOf<R>()
    for (item in this) {
        destination += transform(item)
    }
    return destination
}
```