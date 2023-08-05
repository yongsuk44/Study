# Item 51 : 성능 중심 최적화 시 원시 타입(primitives type) 고려

Kotlin에서는 원시 타입을 직접 선언할 수 없지만, 내부에서는 최적화로 사용됩니다.

원시 타입은 객체에 비해 가볍고 빠르기 때문에, 많은 양의 데이터를 처리할 때 원시 타입 배열을 사용하는 것이 중요한 최적화가 될 수 있습니다.

그러나, Kotlin에서 일반적인 컬렉션(`List`, `Set`)은 제네릭 타입입니다. 
원시 타입은 제네릭 타입으로 사용될 수 없으므로, 대신에 래핑된 타입을 사용하게 됩니다.

이는 대부분의 경우에는 적절한 해결책이 될 수 있지만, 코드 성능 중심 부분에서는 메모리 사용량이 적고 처리가 더 효율적인 `IntArray`나 `LongArray`와 같은 원시 타입 배열을 사용하는 것이 좋습니다.

아래 예시는 `IntArray` 1M개와 `List<Int>` 1M개를 비교한 것입니다.  
측정을 해보면 `IntArray`는 0.4GB, `List<Int>`는 2GB를 사용하고 이는 5배 정도의 차이가 발생됩니다.  
또한 성능에서도 차이가 발생되며 `IntArray`가 `List<Int>`보다 약 25% 빠릅니다.

```kotlin
open class InlineFilterBenchmark {
    lateinit var list: List<Int>
    lateinit var array: IntArray
    
    @Setup
    fun setup() {
        list = (0..1000000).toList()
        array = (0..1000000).toIntArray()
    }
    
    @Benchmark
    fun averageOnIntList(): Double = list.average()
    
    @Benchmark
    fun averageOnIntArray(): Double = array.average()
}
```

정리하면 코드 성능 중심 최적화에서 원시 타입과 원시 타입 배열을 사용하여 메모리 효율과 성능을 향상시킬 수 있습니다.
하지만 대부분의 경우에는 리스트 대신 원시 타입 배열을 기본으로 사용할 만큼 중요한 부분은 아닙니다.

리스트는 더 직관적이고 자주 사용되기에 대부분의 경우에는 리스트를 사용하는 것이 좋습니다.
성능 중심 부분을 최적화해야 하는 경우에만 이러한 방법이 있다는 것을 알고 있으면 됩니다.