'contract'와 'visibility'는 개발자들 사이의 일종의 합의이지만, 이는 거의 대부분 개발자들에 의해 위반된다.  
기술적으로 단일 프로젝트 내의 모든 것은 해킹이 가능하며, 다음과 같이 리플렉션(reflection)을 통해 원하는 모든 것을 열고 사용 할 수 있다.

```kotlin
class Employee {
    private val id: Int = 2

    override fun toString() = "User(id=$id)"

    private fun privateFunction() {
        println("Private function called")
    }
}

fun callPrivateFunction(employee: Employee) {
    employee::class.declaredMemberFunctions
        .first { it.name == "privateFunction" }
        .apply { isAccessible = true }
        .call(employee)
}

fun changeEmployeeId(employee: Employee, newId: Int) {
    employee::class.java.getDeclaredField("id")
        .apply { isAccessible = true }
        .set(employee, newId)
}

fun main() {
    val employee = Employee()
    callPrivateFunction(employee)   // Prints: "Private function called"
    
    changeEmployeeId(employee, 1)
    println(employee)               // Prints: "User(id=1)"
}
```

기술적으로 가능한 행동을 할 수 있지만, 그것이 항상 바람직한 것은 아니다.  
위 예시를 보면, 'private' 프로퍼티와 함수의 이름과 같은 구현 세부사항에 의존하고 있는 것을 볼 수 있다.  
이들은 'contract'의 일부가 아니며, 언제든지 변경될 수 있기에 이는 잠재적인 위험을 가지고 있다.

'contract'를 'warranty'와 같다고 생각해야 한다.  
소프트웨어 개발에서 정의된 'contract'를 준수하는 한 코드는 보호를 받을 수 있다.  
그러나, 개발자가 이를 무시하고 자신의 방식대로 시스템을 변경하거나 확장할 때 생기는 부작용은 오로지 그 개발자의 책임이다.

## Contracts are inherited

클래스를 상속하거나 다른 라이브러리에서 인터페이스를 확장할 때 'contract'를 준수해야 한다.  

예를 들어, 모든 클래스는 'equals'와 'hashCode' 메소드를 가진 'Any'를 확장한다.  
이 둘은 잘 정립된 'contract'를 가지고 있으며, 이를 준수하지 않으면 객체가 올바르게 작동하지 않을 수 있다.  
만약, 'hashCode'가 'equals'와 일관성이 없을 때, 'HashSet'과 같은 컬렉션을 사용할 때 문제가 발생할 수 있다.

```kotlin
class Id(val id: Int) {
    override fun equals(other: Any?) = 
        other is Id && other.id == id
}

val set = mutableSetOf(Id(id = 1))
set.add(Id(id = 1))
set.add(Id(id = 1))
println(set.size)       // Prints: 3
```
