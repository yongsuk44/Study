함수 또는 프로퍼티를 로컬•최상위 변수가 아닌, '리시버'로부터 가져온 것임을 강조하고 싶을 떄 'this' 키워드를 사용하여 명시적으로 참조할 수 있다.

```kotlin
class User: Person() {
    private var beersDrunk: Int = 0
    
    fun drinkBeers(num: Int) {
        this.beersDrunk += num
    }
}
```

마찬가지로, 'extension receiver'를 명시적으로 참조하여 해당 객체의 사용을 더 명확히 할 수 있다.   
아래는 리시버를 명시적으로 참조하지 않고 작성된 퀵소트 구현이다.

```kotlin
fun <T: Comparable<T>> List<T>.quickSort(): List<T> {
    if (size < 2) return this
    val pivot = first()
    val (smaller, bigger) = drop(1).partition { it < pivot }
    return smaller.quickSort() + pivot + bigger.quickSort()
}
```

그리고 아래는 리시버를 명시적으로 참조하여 작성된 퀵소트 구현이다.

```kotlin
fun <T: Comparable<T>> List<T>.quickSort(): List<T> {
    if (size < 2) return this
    val pivot = this.first()
    val (smaller, bigger) = this.drop(1).partition { it < pivot }
    return smaller.quickSort() + pivot + bigger.quickSort()
}
```

위 두 함수 사용법은 동일하다.

```kotlin
listOf(3, 2, 5, 1, 6).quickSort() // [1, 2, 3, 5, 6]
listOf("c", "a", "b").quickSort() // [a, b, c]
```

### Many receivers

하나의 코드 블록 내에서 여러 리시버에 대한 접근이 필요한 경우, 각 객체를 명시적으로 구분하지 않으면 어떤 객체의 메서드나 프로퍼티에 접근하고 있는지 구분하기 어려울 수 있다.
이는 'apply', 'with', 'run'과 같은 스코프 함수를 사용할 때 더 구분하기 어려워 진다.

위 스코프 함수 내부에서 'this'는 스코프 객체를 가리키며, 스코프 함수가 중첩될 경우 'this'가 정확히 어떤 객체를 가리키는지 혼란스러울 수 있다.  
이런 경우 명시적으로 리시버를 참조하여 코드를 작성하면, 작성하려는 의도를 더 명확하게 할 수 있다.

```kotlin
class Node(val name: String) {

    fun create(name: String): Node? = Node(name)
    
    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply { print("Created $name") }
}

fun main(){
    val node = Node("parent") 
    node.makeChild("child")
}
```

위 코드의 실행 결과를 'Created parent.child'로 예상하겠지만, 실제 결과는 'Created parent'이다.  
이런 결과가 나온 이유를 확인하기 위해, 'name' 앞에 리시버를 명시적으로 사용해보면 다음과 같다.

```kotlin
class Node(val name: String) {
    
    fun create(name: String): Node? = Node(name)
    
    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply { print("Created ${this.name}") } // Compilation Error
    
}
```

문제점은 'apply' 내부 'this'의 타입이 'Node?'이기 때문에 메서드를 직접 사용할 수 없다.  
따라서 'Safe-call'을 사용하여 'non-null'일 때, 메서드나 프로퍼티에 접근하도록 처리 해야 한다.

```kotlin
class Node(val name: String) {
    fun create(name: String): Node? = Node(name)
    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply { print("Created ${this?.name}") }
        
}

fun main(){
    val node = Node("parent") 
    node.makeChild("child") // "Created parent.child" 출력
}
```

위 코드는 'apply'의 잘못된 사용 사례로, 'apply' 대신 'also'를 사용하여 함수의 리시버 객체를 명시적으로 참조하도록 강제하여 이러한 문제를 방지할 수 있다.  
이처럼 'also'와 'let'은 리시버를 명시적으로 사용하여, 추가적인 작업을 수행하거나 'nullable' 값에 대한 작업을 수행할 때 적절하다.

```kotlin
class Node(val name: String) {
    
    fun create(name: String): Node? = Node(name)
    
    fun makeChild(childName: String) =
        create("$name.$childName")
            .also { print("Created ${it?.name}") }
}
```

코드 구조에 따라 여러 수준의 리시버가 존재할 수 있으며, 이는 'label'을 통해 명시적으로 리시버를 구분하여 참조할 수 있다.

'label'이 없는 리시버는 가장 가까운 스코프의 리시버를 참조하며, 일반적으로 현재 스코프 내에서 작업하고 있는 리시버 객체를 의미한다.  
반대로 외부 리시버에 접근이 필요한 경우, 'label'을 사용하여 명시적으로 리시버를 참조할 수 있다.

```kotlin
class Node(val name: String) {
    
    fun create(name: String): Node? = Node(name)
    
    fun makeChild(childName: String) =
        create("$name.$childName").apply { 
            print("Created ${this?.name} in ${this@Node.name}")
        }
}

fun main(){
    val node = Node("parent") 
    node.makeChild("child") // Created parent.child in parent
}
```

위와 같이 'direct receiver'의 사용은 코드 내에서 어떤 객체의 메서드나 프로퍼티를 참조하고 있는지를 명확하게 표현하는데 중요한 역할을 한다.  
이렇게 함으로써, 코드의 오류를 예방하고 가독성을 향상시킬 수 있다.