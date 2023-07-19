# Item 37: 데이터 번들 표현 시 데이터 Modifier 사용

때때로 데이터 번들 전달이 필요할 수 있습니다. 이럴 때 개발자들은 데이터 클래스(`data class`)를 사용합니다.

이들은 데이터 수정자(Modifier)가 있는 클래스로 이를 보통 데이터 모델 클래스에 빠르게 도입합니다.

```kotlin
data class Player(
    val id: Int,
    val name: String,
    val points: Int
)

val player = Player(1, "Player 1", 1000)
```

데이터 수정자 추가 시 아래와 같은 유용한 함수들이 제공됩니다.

- `toString`
- `equals`와 `hashCode`
- `copy`
- `componentN`

`toString`은 클래스 이름, 기본 생성자 속성의 이름과 값을 출력합니다.
이는 로깅과 디버깅에 유용하게 사용될 수 있습니다.

    print(player) // Player(id=1, name=Player 1, points=1000)

`equals`는 기본 생성자 속성이 같은지 확인합니다. `hashCode`는 이와 일관성이 있습니다. (#Item 38)

    println(player == Player(1, "Player 1", 1000)) // true
    print(player == Player(1, "Player 2", 1000)) // false

`copy`는 불변 데이터 클래스에 유용하게 사용됩니다.

기본적으로 각 기본 생성자 속성이 동일한 값을 가진 새로운 객체를 만들지만, 명명된 인수를 사용하여 각각의 값을 변경할 수 있습니다.

```kotlin
val newObj = player.copy(name = "Player 22")
```

`copy` 메서드는 데이터 수정자 덕분에 생성되는 다른 메서드와 마찬가지로 내부에서 생성되기에 외부에 노출되지 않습니다.

만약 볼 수 있다면, 다음과 같이 보여질 것입니다.

```kotlin
fun copy(
    id: Int = this.id,
    name: String = this.name,
    age: Int = this.age
) = Player(id, name, age)
```

`componentN`함수들(component1, component2 등)은 위치 기반의 비 구조화를 허용하며 다음과 같이 사용됩니다.

```kotlin
val (id, name, points) = player
```

Kotlin에서의 비구조화는 `componentN`를 사용한 변수 정의로 직접 번역되므로 위 코드는 다음과 같이 컴파일 됩니다.

```kotlin
val id = player.component1()
val name = player.component2()
val point = player.component3()
```

이러한 접근 방식은 장단점이 존재합니다.

가장 큰 장점은 변수의 이름을 원하는 방식으로 작성할 수 있습니다. 또한 `componentN`을 제공하는 모든 것을 비구조화 할 수 있습니다.

`List`와 `Map.Entry`도 `componentN`을 제공합니다. 아래는 그 예시 입니다.

```kotlin
val visited = listOf("China", "Japan", "Korea")
val (first, second, third) = visited

println("$first, $second, $third")
// China, Japan, Korea

val trip = mapOf(
    "China" to "Tianjin",
    "Japan" to "Tokyo",
    "Korea" to "Seoul"
)

for ((country, city) in trip) {
    println("We loved $city in $country")
    // We loved Tianjin in China
    // We loved Tokyo in Japan
    // We loved Seoul in Korea
}
```

단점으로는 예기치 못한 상황이 생길 수 있습니다.

데이터 클래스에서 요소의 순서가 변경되면 모든 비구조화를 조정해주어야 합니다.
또한 순서를 혼동해서 잘못된 비구조화를 하기에 쉽습니다.

```kotlin
data class FullName(
    val firstName: String,
    val secondName: String,
    val lastName: String
)

val elon = FullName("Elon", "Reeve", "Musk")
val (name, surname) = elon
println("It is $name $surname!")
```

비구조화 시 주의해서 작업을 해야 하며 데이터 클래스 기본 생성자 속성과 동일한 이름을 사용하는 방법이 실수를 줄일 수 있는 방법입니다.

만약 잘못된 순서가 있다면 Tool에서 경고가 표시되므로 이 경고를 바탕으로 수정할 수 있습니다.

## tuples 대신 데이터 클래스 권장

데이터 클래스는 일반적으로 튜플이 제공하는 것보다 더 많은 것을 제공합니다.

코틀린의 튜플은 `Serializable`이며 커스텀 `toString` 메서드가 있는 일반 데이터 클래스입니다.

```kotlin
public data class Pair<out A, out B>(
    public val first: A,
    public val second: B
) : Serializable {
    override fun toString(): String = "($first, $second)"
}

public data class Triple<out A, out B, out C>(
    public val first: A,
    public val second: B,
    public val third: C
) : Serializable {
    override fun toString(): String = "($first, $second, $third)"
}

```

현재 Kotlin에 남아있는 마지막 튜플이 위 2가지 `Pair`와 `Triple`입니다.

다음과 같이 괄호와 일련의 타입으로 튜플을 정의할 수 있습니다. : `(Int, String, String, Long)`

이는 데이터 클래스와 동일하게 동작하지만, 위와 같은 튜플 타입이 어떤것을 나타내는지 어렵기 떄문에 가독성에 문제가 생길 수 있습니다.
튜플을 사용하는것 편할 수 있지만, 데이터 클래스를 사용하는 것이 더 좋습니다.

위 2가지 튜플은 아래와 같은 로컬 목적으로 사용되기 때문에 현재까지 남아 있을 수 있었습니다.

### 값의 이름을 바로 지정할 때

```kotlin
val (description, color) = when {
    degrees < 5 -> "cold" to Color.BLUE
    degrees < 23 -> "mild" to Color.YELLOW
    else -> "hot" to Color.RED
}
```

### 예측할 수 없는 집계를 나타날 때 - 이들은 표준 라이브러리 함수에서 흔하게 볼 수 있습니다.

```kotlin
val (odd, even) = numbers.partition { it % 2 == 1 }
val map = mapOf(1 to "San Francisco", 2 to "Seoul")
```

그 외의 경우에는 데이터 클래스를 사용하는 것이 좋습니다.

아래는 이름과 성을 파싱하는 하뭇가 필요하다는 가정으로 이름과 성을 `Pair<String, String>`으로 표현한 예시 입니다.

```kotlin
fun String.parseName(): Pair<String, String>? {
    val indexOfLastSpace = this.trim().lastIndexOf(' ')

    if (indexOfLastSpace < 0) return null
    val firstName = this.take(indexOfLastSpace)
    val lastName = this.drop(indexOfLastSpace)

    return Pair(firstName, lastName)
}

val fullName = "Kim YongSuk"
val (firstName, lastName) = fullName.parseName() ?: return
```

이는 `Pair<String, String>`이 이름을 나타내는 것이 명확하지 않다는 문제가 있을 수 있습니다.
또한 값의 순서가 명확하지 않아 성이 먼저 올 수 있다고 생각할 수 있습니다.

```kotlin
val fullName = "Kim yongsuk"
val (lastName, firstName) = fullName.parseName() ?: return
print("Name $firstName") // Name yongsuk
```

이처럼 사용에 있어 안전하고 함수의 가독성을 높이려면 데이터 클래스를 사용하는 것이 좋습니다.

```kotlin
data class FullName(
    val firstName: String,
    val lastName: String
)

fun String.parseName(): FullName? {
    val indexOfLastSpace = this.trim().lastIndexOf(' ')

    if (indexOfLastSpace < 0) return null
    val firstName = this.take(indexOfLastSpace)
    val lastName = this.drop(indexOfLastSpace)

    return FullName(firstName, lastName)
}

val fullName = "Kim yongsuk"
val (firstName, lastName) = fullName.parseName() ?: return
```

데이터 클래스 사용 시 다음과 같이 함수 사용성을 높일 수 있습니다.

- 함수의 반환 타입이 명확함
- 반환 타입이 더 짧고 전달하기 쉬움
- 사용자가 데이터 클래스에 설명된 것과 다른 이름으로 변수를 비구조화하면 경고가 발생
