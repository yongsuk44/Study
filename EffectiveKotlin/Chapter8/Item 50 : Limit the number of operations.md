컬렉션을 처리하는 모든 방법에는 그 만큼의 비용이 든다.  
표준 컬렉션 처리에 있어서, 이는 주로 요소들을 다시 한 번 순회하며 그 과정에서 추가적인 컬렉션을 생성하는 것을 의미한다.  
'Sequence' 처리에서는 전체 'Sequence'를 래핑하는 객체와 각 작업을 유지하기 위한 별도의 객체가 필요하게 된다.
이런 처리 과정은 처리해야 할 요소의 수가 많아질수록 비용이 기하급수적으로 증가한다.  
이러한 이유로 컬렉션 처리 단계의 수를 제한하며, 주로 복합 연산을 통해 이를 해결하는 것이 좋다.

예를 들어, 'null'이 아닌 것들을 필터링한 후에 'Non-nullable' 타입으로 캐스팅하는 대신, 'filterNotNull'을 사용할 수 있다.  
또는 매핑한 후에 'null'을 필터링하는 대신, 'mapNotNull'을 사용하여 한 단계에서 처리할 수 있다.

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

가장 큰 문제는 위와 같은 변경의 중요성을 잘못 이해하는 것이 아닌, 어떤 컬렉션 처리 함수를 사용해야 하는 지식이 부족한 것이다.  
이는, 더 나은 대안을 제안해 주는 경고 메시지들 덕분에, 더 효율적인 코드로 개선할 수 있도록 도움을 준다.

그럼에도, alternative composite 연산들을 알아두는 것이 좋으며, 아래는 일반적인 몇 가지 함수 호출과 연산 횟수를 제한하는 대체 방법들의 표이다. 

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