# Flow building

## Flow from raw values

`Flow`를 생성하는 가장 간단한 방법은 `flowOf` 함수를 사용하는 것입니다.  
`flowOf`는 주어진 원시 값들로 `Flow`를 생성하며 이는 리스트를 생성하는 `listOf`와 유사한 원리로 동작합니다.

```kotlin
flowOf(1, 2, 3, 4, 5).collect { print(it) }
// 12345
```

값이 없는 Flow를 생성하려면 `emptyFlow`를 사용할 수 있습니다.

```kotlin
emptyFlow<Int>().collect { print(it) }
// nothing
```

------------------------------------------------------------------

## Converters

`asFlow`는 `Iterable`, `Iterator`, `Sequence`와 같은 컬렉션 형태의 데이터 구조를 `Flow`로 변환하는 데 사용됩니다.

```kotlin
listOf(1, 2, 3, 4, 5)
    // setOf(1, 2, 3, 4, 5)
    // sequenceOf(1, 2, 3, 4, 5)
    .asFlow()
    .collect { print(it) }
// 12345
```
