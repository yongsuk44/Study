# Item 40: equals 계약 준수

Kotlin에서 모든 객체는 `Any`를 상속합니다.

`Any`는 몇 가지 메서드와 그에 해당하는 계약(Contract)을 포함하고 있습니다.

- `equals`
- `hashCode` 
- `toString`

위 메서드들은 Kotlin에서 중요한 역할을 하며, Java가 처음 생겨났을 때 부터 정의되어 왔기 때문에 많은 객체와 함수들이 이 메서드들의 계약에 의존하고 있습니다.
위 계약을 어기게 되면 일부 객체나 함수가 제대로 작동되지 않을 수 있습니다.

## 동등성(Equality)

Kotlin에서는 2가지 타입의 동등성이 존재합니다.

### 구조적 동등성(Structural Equality)

`equals` 메서드나 `==` 연산자를 통해 확인할 수 있습니다. 이는 `equals` 메서드에 기반을 두고 있습니다.

`a == b`는 `a != null`이면 `a.equals(b)`로 변환되며, 그렇지 않으면 `a?.equals(b) ?: (b === null)`로 변환됩니다.

### 참조 동등성(Referential Equality)

`===` 연산자를 통해 확인할 수 있으며 두 측이 같은 객체를 가리킬 때 `true`를 반환합니다.

`equals`는 모든 클래스의 `super class`인 `Any`에서 구현되므로, 어떤 두 객체의 동등성도 체크할 수 있습니다.

```kotlin
open class Animal
class Book
class Cat: Animal()

Animal() == Book() // Error : Operator == cannot be applied to Animal and Book
Animal() === Book() // Error : Operator === cannot be applied to Animal and Book

Animal() == Cat() // OK, because Cat is a subclass of Animal
Animal() === Cat() // OK, because Cat is a subclass of Animal
```
