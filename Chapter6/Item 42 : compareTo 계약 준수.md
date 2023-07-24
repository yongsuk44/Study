# Item 42: compareTo 계약 준수

`compareTo()`는 `Any` 클래스에 속하지 않으며, Kotlin에서는 수학적인 부등호로 변환되는 연산자입니다.

```kotlin
obj1 > obj2 // Translates to obj1.compareTo(obj2) > 0
obj1 < obj2 // Translates to obj1.compareTo(obj2) < 0
obj1 >= obj2 // Translates to obj1.compareTo(obj2) >= 0
obj1 <= obj2 // Translates to obj1.compareTo(obj2) <= 0
```

`compareTo()`는 `Comparable<T>` 인터페이스에 위치하고 있으며 객체가 `Comparable<T>` 인터페이스를 구현하거나 `compareTo()` 연산자 메서드를 갖고 있다면 해당 객체는 자연스러운 순서를 가지는 객체로 볼 수 있습니다.

이러한 순서는 아래와 같은 특징을 가집니다.

### 반대 대칭성 (Anti-symmetric)

`a >= b`, `b >= c` 이면 `a == b` 입니다. 따라서 비교와 동등성은 서로 일관성을 가져야 합니다.

### 전이성 (Transitive)

`a >= b`, `b >= c` 이면 `a >= c` 입니다. 이와 마찬가지로 `a > b`, `b > c` 이면 `a > c` 입니다.

위 특성을 활용하지 않게되면 일부 정렬 알고리즘에서 요소의 정렬이 무한히 걸릴 수 있기에 중요합니다.

### 연결성 (Connex)

`a >= b`이거나 `b >= a`와 같이 모든 두 요소 사이에 관계가 있어야 합니다.

`compareTo()`는 `Int`를 반환하고, 모든 `Int`는 양수, 음수, 0인 것을 보장합니다.
이 성질은 두 요소 간에 관계가 없다면 퀵 정렬이나 삽입 정렬과 같은 일반적인 알고리즘을 사용할 수 없기에 중요합니다.