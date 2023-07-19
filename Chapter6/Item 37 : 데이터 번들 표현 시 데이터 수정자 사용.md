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

    print(player)

`equals`는 기본 생성자 속성이 같은지 확인합니다. `hashCode`는 이와 일관성이 있습니다. (#Item 38)

    println(player == Player(1, "Player 1", 1000))
    print(player == Player(1, "Player 2", 1000))

`copy`는 불변 데이터 클래스에 유용하게 사용됩니다.

기본적으로 각 기본 생성자 속성이 동일한 값을 가진 새로운 객체를 만들지만, 명명된 인수를 사용하여 각각의 값을 변경할 수 있습니다.

    val newObj = player.copy(name = "Player 22")

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

    val (id, name, points) = player

Kotlin에서의 비구조화는 `componentN`를 사용한 변수 정의로 직접 번역되므로 위 코드는 다음과 같이 컴파일 됩니다.

    val id = player.component1()
    val name = player.component2()
    val point = player.component3()

이러한 접근 방식은 장단점이 존재합니다.

가장 큰 장점은 변수의 이름을 원하는 방식으로 작성할 수 있습니다. 또한 `componentN`을 제공하는 모든 것을 비구조화 할 수 있습니다.

`List`와 `Map.Entry`도 `componentN`을 제공합니다. 아래는 그 예시 입니다.

```kotlin
val visited = listOf("China", "Japan", "Korea")
val (first, second, third) = visited

println("$first, $second, $third")

val trip = mapOf(
    "China" to "Tianjin",
    "Japan" to "Tokyo",
    "Korea" to "Seoul"
)

for ((country, city) in trip) {
    println("We loved $city in $country")
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