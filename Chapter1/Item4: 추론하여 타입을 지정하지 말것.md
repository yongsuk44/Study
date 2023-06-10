## Kotlin의 타입 추론
- 할당의 추론된 타입이 오른쪽 타입이 정확한 타입
- 상위 클래스나 인터페이스로 설정하지 않는다.

```kotlin
open class Animal
class Zebra : Animal()

fun main() {
    var animal = Zebra()
    animal = Animal() // Error: Type mismatch

    var animal: Animal = Zebra()
    animal = Animal()
}
```
---
### 추론된 타입의 노출
- 외부 라이브러리나 다른 모듈을 제어할 수 없는 경우, 추론된 타입의 노출은 에러를 발생시킬 수 있다. 

```kotlin
interface CarFactory {
    fun produce(): Car
}

val DEFAULT_CAR: Car = Ferrari()
```

최초 코드 작성 시 위와 같이 작성하였지만,
나중에 `DEFAULT_CAR`는 `Car`로 명시적으로 지정되어 있어 리턴 타입을 아래와 같이 수정하게 되었다.

``` kotlin
interface CarFactory {
    fun produce(): DEFAULT_CAR
}
```

이후에 협업하는 개발자가 `DEFAULT_CAR`에 **타입 추론에 의해** 자동으로 타입이 지정될 것으로 생각하여  
`DEFAULT_CAR`에서 명시된 `Car` 타입을 제거하였다.

```kotlin
val DEFAULT_CAR = Ferrari()
```

그러면 이후에는 `Ferrari()`밖에 생산하지 못하는 문제가 생길 수 있다.  
만약 우리가 직접 수정할 수 있다면 크게 문제가 없지만,  
외부 라이브러리나 모듈을 사용하는 경우 외부 개발자에게 문의하여 시간에 비용이 든다.

---

### 정리
- 타입에 대해 확신이 없으면 타입을 명시해야 한다.   
- 보안을 위해, 외부 API에서는 항상 타입을 명시해야 한다.
- 추론된 타입은 너무 제한적일 수도 있고, 프로젝트를 진행하면서 너무 쉽게 변경될 수도 있음을 알고 있어야 한다.