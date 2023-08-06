# Item 52: mutable 컬렉션 사용 시기

immutable 컬렉션에 요소 추가 시 새로운 컬렉션을 생성하고 모든 요소를 추가해야하는 비용이 발생하기에,
mutable 컬렉션을 사용하는 것이 성능 측면에서 좋습니다.

```kotlin
public operator fun <T> Iterable<T>.plus(element: T): List<T> {
    if (this is Collection) return this.plus(element)
    
    val result = ArrayList<T>()
    result.addAll(this)
    result.add(element)
    return result
}
```

개발자들은 immutable 컬렉션이 안전성 측면에 이점을 가지고 있다는 것을 알고 있지만 이러한 이점은 주로 동기화 혹은 캡슐화가 중요한 상황에서 적용됩니다.
로컬 변수와 같이 위 상황이 중요하지 않은 경우에는 오히려 mutable 컬렉션을 사용하는 것이 더 좋습니다.

아래는 Kotlin 표준 라이브러리의 모든 컬렉션 처리 함수에서는 내부에서 mutable 컬렉션을 사용하여 성능을 향상시키고 있는 것을 알 수 있습니다.

```kotlin
inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R> {
    val destination = ArrayList<R>(if (this is Collection<*>) this.size else 10)
    for (item in this) destination.add(transform(item))
    return destination
}
```