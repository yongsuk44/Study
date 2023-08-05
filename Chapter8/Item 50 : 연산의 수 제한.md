# Item 50: 연산의 수를 제한

컬렉션 연산 시 ‘모든 요소를 순회하는 작업’과 ‘내부에서 추가적인 컬렉션을 생성으로 인한 메모리 사용량 증가’ 등과 같이 연산하는데에 비용이 발생됩니다.

Sequence 처리의 경우 전체 `Sequence`를 감싸는 객체와 작업을 유지하는 또 다른 객체가 필요하며 만약 처리되는 요소의 수가 많을 경우, 위 2가지 객체에 대한 비용이 상당히 발생될 수 있습니다.

따라서 컬렉션 연산 단계의 수를 제한하고 이를 복합 연산을 통해 해결하도록 하는 것이 좋습니다.

예를 들어 `null`이 아닌 것을 필터링한 다음 `non-nullable` 타입으로 캐스팅하는 `filterNotNull` 등이 있습니다.
또는 맵핑 후 `null`을 필터링하는 대신에 `mapNotNull`을 바로 사용할 수 있습니다.

```kotlin
class Student(val name: String?)

// Works
fun List<Student>.getNames(): List<String> = this
    .map { it.name }
    .filter { it != null }
    .map { it!! }

// Better
fun List<Student>.getNames(): List<String> = this
    .map { it.name }
    .filterNotNull()

// Best
fun List<Student>.getNames(): List<String> = this
    .mapNotNull { it.name }
```

아래는 일반적으로 사용되는 몇 가지 함수 호출과 그에 대안하는 방법들의 표 입니다.

| instead of                                                                                      | use                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| .filter { it != null } <br/> .map { it!! }                                                      | .filterNotNull()                                         |
| .map { <Transformation> } <br/> .filterNotNull()                                                | .mapNotNull { <Transformation> }                         |
| .map { <Transformation> } <br/> .joinToString()                                                 | .joinToString(separator = "") { <Transformation> }       |
| .filter { <Predicate 1> } <br/> .filter { <Predicate 2> }                                       | .filter { <Predicate 1> && <Predicate 2> }               |
| .filter {it is Type } <br/> .map { it as Type }                                                 | .filterIsInstance<Type>()                                |
| .sortedBy { <Key 2> } <br/> .sortedBy { <Key 1> }                                               | .sortedWith(compareBy({ <Key 1> }, { <Key 2> }))         |
| listOf(...).filterNotNull()                                                                     | listOfNotNull(...)                                       |
| .withIndex() <br/> .filter { (index, elem) -> <Predicate using index> } <br/> .map { it.value } | .filterIndexd { index, elem -> <Predicate using index> } |