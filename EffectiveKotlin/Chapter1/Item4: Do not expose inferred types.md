Kotlin의 타입 추론은 개발자가 타입을 명시적으로 선언하지 않아도, 컴파일러가 적절한 타입을 자동으로 결정해주는 기능이다.  
하지만 타입 추론을 사용할 때 주의해야 할 점으로 변수에 값을 할당할 때, 컴파일러는 오른쪽 항의 정확한 타입을 변수의 타입으로 추론한다.  
이는 할당된 값이 어떤 인터페이스나 상위 클래스의 타입을 구현하더라도, 실제 할당된 객체의 타입을 기준으로 타입이 결정됨을 의미한다.

만약 'inferred type'이 너무 제한적인 타입으로 결정될 때, 개발자는 타입을 명시적으로 선언하여 문제를 해결할 수 있다. 

```kotlin
open class Animal
class Zebra : Animal()

fun main() {
    var animal = Zebra()
    animal = Animal() // Error: Type mismatch

    var animal: Animal = Zebra() // Specify type explicitly
    animal = Animal() // OK
}
```

'inferred type'이 위험한 시나리오는 외부 라이브러리나 모듈을 사용할 때 이다.   
이런 경우, 라이브러리의 업데이트나 변경에 따라 'inferred type'이 예상치 못하게 변경될 수 있으며, 이로 인해 코드가 불안정해 질 수 있다.

예를 들어, 자동차 공장을 나타내기 위해 사용되는 인터페이스가 있다고 가정해보자. 

```kotlin
interface CarFactory {
    fun produce(): Car
}
```

대부분의 공장에서는 기본 자동차가 생산되므로 아무런 값이 지정되지 않았을 때, 기본 자동차를 생산하도록 할 수 있다.

```kotlin
val DEFAULT_CAR: Car = Fiat126P()
```

그 후, `DEFAULT_CAR`는 어차피 `Car` 타입으로 결정되었기에 타입을 생략했다.

``` kotlin
interface CarFactory {
    fun produce(): DEFAULT_CAR
}
```

시간이 지나, 나중에 어떤 개발자가 `DEFAULT_CAR`의 타입을 추론할 수 있다고 생각하여 코드를 수정하였다. 

```kotlin
val DEFAULT_CAR = Fiat126P()
```

이 후, 모든 자동차 공장은 `Fiat126P`만 생산되기에 좋지 않은 결과가 발생했다.

만약 이 인터페이스를 직접 정의했다면, 이 문제는 쉽게 발견되어 수정되었을 것이다.  
하지만 이것이 라이브러리나 외부 모듈의 일부라면 이를 해결하기 위한 시간과 비용이 발생하게 된다.