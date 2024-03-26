Kotlin에서는 'primitive' 타입을 직접 선언할 수 없다.  
그러나, 성능 최적화 차원에서, 'primitive' 타입은 내부적으로 사용되고 있다.  
이 최적화 전략은 'Item 45'에서도 언급된 것처럼 중요한 개념이며, 이들은 다음과 같은 이유로 성능 향상에 기여한다. 

- 모든 객체는 추가적인 메모리 부담을 가지지만, 'primitive' 타입은 가볍기 때문에 부담이 적다.
- 값에 접근하기 위해 accessors를 사용하는 것은 추가적인 처리 비용을 발생시키지만, 'primitive' 타입은 이러한 비용이 없이 더 빠른 접근이 가능하다.

따라서 대량의 데이터를 처리할 때 'primitive' 타입을 사용하는 것은 중요한 최적화 방법이 될 수 있다.  
Kotlin에서는 'List'나 'Set'과 같은 일반적인 컬렉션들이 제네릭 타입으로 되어 있다.  
'primitive' 타입은 이러한 제네릭 타입에 사용될 수 없기에, 래핑된 타입을 사용하게 된다.  
이 방법은 대부분의 경우에 유용하고 표준 컬렉션을 이용한 처리를 더 쉽게 만든다.  
그러나, 코드의 성능이 매우 중요한 부분에서는 메모리 사용량이 적고 처리 효율이 더 높은 'IntArray' 또는 'LongArray'와 같은 'primitive' 타입 배열의 사용을 고려해야 한다.

How much lighter arrays with primitives are? Let’s say that in Kotlin/JVM we need to hold 1 000 000 integers, and we can either choose to keep them in IntArray or in List<Int>. When you make measurements, you will find out that the IntArray allocates 400 000 016 bytes, while List<Int> allo- cates 2 000 006 944 bytes. It is 5 times more. If you optimize for memory use and you keep collections of types with primitive analogs, like Int, choose arrays with primitives.

Kotlin/JVM에서 100만 개의 정수를 저장해야 하는 상황을 가정해보자.  
이를 위해 'IntArray'와 'List<Int>' 중 하나를 선택할 때, 실제 측정을 해보면, 'IntArray'의 경우 400,000,016 바이트를 사용하는 반면, 'List<Int>'는 2,000,006,944 바이트를 할당한다. 
즉, 'List<Int>'의 메모리 사용량은 'IntArray' 보다 5배나 더 많다. 따라서, 메모리 사용 최적화가 중요한 경우, 'Int'와 같이 'primitive' 타입 대응이 가능한 타입의 컬렉션을 다룰 떄는 'primitive' 타입 배열을 선택하는 것이 좋다.

또한, 성능에도 차이가 있으며, 같은 1,000,000개의 숫자 컬렉션에서 평균을 계산할 때, 'primitive' 타입의 배열을 사용하면 약 25% 더 빠르다.

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

정리하면 코드 중 성능이 매우 중요한 부분에서는 'primitive' 타입과 'primitive' 타입 배열이 효율적인 최적화 수단이 될 수 있음을 알 수 있다.  
이 방법들은 메모리 할당량을 줄이고 처리 속도를 향상시키는 이점을 제공한다. 그럼에도 불구하고, 'List'를 'primitive' 타입 배열로 대체해야 할 만큼 대부분의 경우에 상당한 개선이 필요하지는 않다.
'List'는 사용하기 더 직관적이기에 자주 사용되므로, 대부분의 상황에서는 리스트를 사용해야 한다.  
단지 성능이 중요한 부분을 최적화해야 할 때만 이러한 방법을 고려하면 된다.
