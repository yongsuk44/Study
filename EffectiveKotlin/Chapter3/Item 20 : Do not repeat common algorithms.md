개발자들은 종종 특정 프로젝트에 국한되지 않고, 일반적인 문제를 해결하려는 코드 패턴이나 알고리즘을 불필요하게 구현하려는 경향이 있다.  
대부분의 불필요한 알고리즘은 비즈니스 로직과 별개로, 수학적 연산이나 컬렉션 처리 같은 일반적인 기능을 수행하거나, 복잡한 알고리즘을 수행하는 등 다양한 형태로 나타난다.

예를 들어, 범위 내에 숫자를 강제하는 아래와 같은 로직을 만들었다고 가정해면, 다음과 같다.

```
val percent = when {
    numberFromUser > 100 -> 100
    numberFromUser < 0 -> 0
    else -> numberFromUser
}
```

하지만, 이는 이미 표준 라이브러리의 `coerceIn`으로 정의되어 있어 직접 구현할 필요가 없다.

```
val percent = numberFromUser.coerceIn(0, 100)
```

그럼에도, 짧지만 반복되는 알고리즘을 추출하는 접근 방식은 개발 프로세스의 여러 측면에서 이점을 제공한다.

1. 프로그래밍 속도 향상 : 여러 단계를 포함하는 알고리즘을 작성하는 것보단, 함수 호출 한 번이 훨씬 간결하기에 코드 작성 시간이 단축된다.
2. 명확한 개념 인식 : 알고리즘이 이름으로 명시되어 있기에, 개발자는 구현을 하나하나 읽지 않고도 해당 기능을 바로 알 수 있다.  
   이는 새로운 개발자들에게는 처음에는 어려울 수 있지만, 장기적으로 보면 큰 이점을 가진다.
3. 비정상 로직 감지 용이성 : 불필요한 정보를 줄임으로써 코드 내 특이점을 파악하기 더 쉬워진다.  
   예를 들어, 정렬 함수 사용 시 'sortedBy'와 'sortedByDescending' 함수들은 구현이 유사하지만, 정렬 방향을 명확하게 알 수 있다.  
   만약 매번 이 로직을 구현해야 한다면, 정렬이 오름차순인지 내림차순인지 혼동하기 쉬울 것이다.  
   또한, 주석이 있더라도 코드를 변경하면서 주석을 업데이트하지 않는 경우가 있을 수 있기에, 신뢰할 수 없는 경우가 더러 있다.
4. 한 번의 최적화로 모든 사용처에서 성능 향상의 이점을 볼 수 있으며, 이는 코드 전반에 걸쳐 일관성과 효율성을 높여준다.

---

## Learn the standard library

대부분의 일반적인 알고리즘은 누군가에 의해 이미 정의되어 있으며, 많은 라이브러리들은 이런 알고리즘들의 집합이다.  
이 중 가장 특별한 것은 표준 라이브러리(stdlib)이다. 이는 주로 확장 함수로 정의된 다양한 유틸리티의 거대한 집합이다.

표준 라이브러리 함수들을 학습하는 것은 까다롭긴 하지만, 매우 가치가 있는 행동이다.  
만약, 표준 라이브러리를 활용하지 않는 개발자는 동일한 문제에 대해 계속해서 새로운 해결책을 개발할 것이다.

아래 예시를 보자.

```
override fun saveCallResult(item: SourceResponse) {
    var sourceList = ArrayList<SourceEntity>()
    items.sources.forEach {
        val sourceEntity = SourceEntity()
        sourceEntity.id = it.id
        sourceEntity.name = it.name
        sourceEntity.description = it.description
        sourceList.add(sourceEntity)
    }

    db.insertSources(sourceList)
}
```

위에서 'forEach' 구문의 사용은, 'for-loop'를 사용하는 것과 비교했을 때 별다른 장점을 제공하지 않아 보인다.  
하지만, 이 코드에서 주목해야할 부분은 하나의 타입에서 다른 타입으로 변환이 이루어지고 있다는 점이다.  
이런 경우에는 'map' 함수를 활용하는 것이 적합하다.

더불어, 'SourceEntity'의 설정 방식은 옛날 방식인 JavaBean 패턴이기에, 이 대신 'Factory-function'이나 'Primary-constructor'를 사용해야 한다.  
혹시라도 JavaBean 패턴을 유지할 필요가 있다면, 적어도 'apply'를 사용하여 단일 객체의 모든 속성을 암시적으로 설정해야 한다.

아래는 간단한 정리를 마친 모습이다.

```
override fun saveCallResult(item: SourceResponse) {
    val sourceEntries = item.sources.map(::sourceToEntry)
    db.insertSources(sourceEntries)
}

private fun sourceToEntry(source: Source) = 
    SourceEntity().apply {
        id = source.id
        name = source.name
        description = source.description
    }
```

---

## Implementing your own utils

모든 프로젝트에는 표준 라이브러리에 없는 특정 알고리즘이 필요한 순간들이 있다.  
예를 들어, 컬렉션에 있는 숫자의 곱을 계산이 필요하다고 가정해보면, 널리 알려진 'fold'를 활용하여 보편적인 유틸리티 함수로 정의하는 것이 좋다.

```
fun Iterable<Int>.product() = fold(1) { acc, i -> acc * i }
```

위와 같이 보편적인 유틸리티 함수를 정의 시, 여러 번 사용될 함수만 정의할 필요는 없다.  
이미 잘 알려진 수학적 개념은 그 이름 자체가 개발자에게 명확하기에, 처음부터 구현해 두는 것이 좋다.  
그러면 미래에 다른 개발자가 해당 함수를 필요로 할 때, 이미 정의되어 있음을 확인하고 큰 도움을 받을 것이다.

주의할 점은 동일한 결과를 얻는 중복 함수를 만드는 것이다.  
각 함수들은 반드시 테스트되어야 하고, 기억되어야 하며, 유지보수 되어야 하기에 비용으로 간주된다.  
따라서 함수를 무분별하게 추가하는 것보단, 필요한 함수가 이미 존재하는지 먼저 검색해야 한다.

'product' 함수처럼 표준 라이브러리의 대부분 함수는 '확장 함수'라는 것을 주목해야 한다.  
개발자들은 'top-level function', 'property delegate' 또는 클래스에 이르기까지 다양한 방법으로 공통 알고리즘을 추출할 수 있다.  
그럼에도 불구하고 확장 함수는 다음과 같은 이유로 가장 좋은 선택지이다.

- 상태 불필요 : 함수는 상태를 가지지 않기에, 부작용이 없는 행동을 표현하는데 적합하다.
- 구체적인 타입 제안 : 'top-level function'에 비해, 확장 함수는 구체적인 타입의 객체에만 제안되기에 사용하기 더 쉽다.
- 직관적인 수정 : 'argument'의 수정 보단, 'extension receiver'의 수정이 더 직관적이다.
- 힌트에서의 발견 용이성 : 객체의 메서드에 비해, 확장 함수는 객체에만 제안되기에 힌트 중에서 찾기 더 쉽다.  
  예를 들어, `"Text".isEmpty()`는 `TextUtils.isEmpty("Text")` 보다 찾기 쉽다.
- 혼동 방지 : 확장 함수는 명확한 호출 컨텍스트를 제공함으로, 메서드 호출 시 '클래스 또는 슈퍼 클래스의 메서드'와 'top-level function' 사이의 혼동 가능성을 줄여준다.