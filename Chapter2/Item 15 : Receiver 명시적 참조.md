# 항목 15: 명확하게 리시버 지정 : 명시적 참조의 중요성

---

receiver(이하 리시버)를 명시적으로 참조하면 `함수`나 `속성`이 로컬 변수나 상위 레벨 변수가 아닌 `객체`로부터 가져오는 것을 명확하게 나타낼 수 있습니다. 

가장 기본적인 상황에서 이는 메소드와 연관된 클래스를 참조한다는 것을 의미합니다.

```kotlin
class User: Person() {
    private var beersDrunk: Int = 0
    
    fun drinkBeers(num: Int) {
        this.beersDrunk += num
    }
}
```

### Extension receiver
명시적으로 extension receiver(이하 확장 리시버)를 참조하여 코드르 작성하는 것이 작성하려는 의도를 더 명확하게 할 수 있습니다. 

```kotlin
fun <T: Comparable<T>> List<T>.quicksort(): List<T> {
    if (size < 2) return this
//    val pivot = first() // 리시버 명시 없이 작성
    val pivot = this.first() // 리시버 명시 작성
    val (smaller, bigger) = drop(1).partition { it < pivot }
    return smaller.quickSort() + pivot + bigger.quickSort()
}
```

## 여러 리시버 활용
 
`apply`, `with`, `run` 함수를 사용하면 여러 리시버의 범위를 갖는 상황에 놓일 수 있는데, 이러한 코드 작성은 위험할 수 있으므로 피하는 것이 좋습니다. 

만약 사용해야 한다면, 명시적 리시버를 사용해 객체를 활용하는 것이 안전합니다.

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName").apply { print("Created $name") }
        
    fun create(name: String): Node? = Node(name)
}

fun main(){
    val node = Node("parent") 
    node.makeChild("child")
}
```

대부분의 사람들은 위 코드를 실행하면 "Created parent.child"라는 결과를 예상할 것입니다.   
하지만 실제 결과는 "Created parent"입니다. 

왜 그럴까요? 이를 파악하기 위해, `name` 앞에 명시적 리시버를 사용해보죠
```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName").apply { print("Created ${this.name}") }
    // Compilation Error
    
    fun create(name: String): Node? = Node(name)
}
```

`apply` 내부에서 `this`의 타입은 `Node?`이기 때문에 메소드를 바로 사용할 수 없습니다.   
그래서 먼저 `(.?)` = Safe Call을 사용하여 이를 처리해야 합니다.

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName").apply { print("Created ${this?.name}") }
        
    fun create(name: String): Node? = Node(name)
}

fun main(){
    val node = Node("parent") 
    node.makeChild("child") // "Created parent.child" 출력
}
```

그러나, 위 코드들은 `apply`의 부적절한 사용 사례입니다.   
만약 `also`를 사용했다면 이러한 문제가 발생하지 않습니다. `also`를 사용하면 함수의 리시버를 명시적으로 참조하게 됩니다.   
일반적으로, 추가적인 연산이나 `null` 가능성이 있는 값에 작업을 수행할 때 `also`나 `let`을 사용하는것이 좋습니다.

### Label Receiver
리시버가 어떤 것을 가리키는지 명확하지 않은 경우에는, 리시버를 피하거나 명시적으로 리시버를 사용해 명확하게 표현해야 합니다.   
라벨이 없는 리시버를 사용하면 가장 가까운 것을 참조하게 됩니다. 그러나 외부의 리시버를 참조하려면, 라벨을 사용해야 합니다.

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName").apply { 
            print("Created ${this.name} in ${this@Node.name}")
        }
        
    fun create(name: String): Node? = Node(name)
}

fun main(){
    val node = Node("parent") 
    node.makeChild("child") // "Created parent.child in parent" 출력
}
```

위 코드에서, `this.name`는 `apply` 블록의 리시버, 즉 새로 생성된 'Node' 객체의 'name'을 참조합니다.   
반면에, `this@Node.name`은 'Node' 클래스의 리시버, 즉 'Node' 객체(부모 노드)의 'name'을 참조합니다.

이렇게 명시적으로 리시버를 사용하면, 참조하고자 하는 리시버가 무엇인지 명확하게 이해할 수 있습니다.   
이 정보는 코드의 오류를 예방하고, 가독성을 향상시키는 중요한 역할을 수행합니다.  
따라서, 리시버가 무엇을 가리키는지 명확하지 않을 때에는 이와 같이 명시적으로 리시버를 사용하는 것이 좋습니다.