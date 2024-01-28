Kotlin의 'Null-safety'는 Kotlin을 특별하게 만드는 주요 기능 중 하나이며, Java에서 흔하게 발생되는 'Null-Pointer Exception'을 방지하는데 큰 도움이 된다.
이처럼 Kotlin에서는 '타입 시스템'을 통해 변수가 'null'이 될 수 있는지를 명시적으로 처리한다.  
즉, Kotlin에서는 'null'이 될 수 있는 변수와 그렇지 않은 변수가 명확하게 구분된다.

그러나, Kotlin과 'Null-safety'가 확실하지 않은 언어(e.g : Java, C) 사이의 상호작용이 발생하게 되면 완전하게 'Null-safety'를 보장할 수 없다.
예를 들어, Java에서 작성된 메서드가 'String' 타입을 반환하다고 가정해보면 Kotlin에서는 어떤 타입으로 변환될까?

만약 메서드가 `@Nullable`으로 주석 처리되어 있다면, 이를 'nullable'로 간주하고 `String?`으로 해석한다.  
`@NotNull`로 주석 처리된 경우에는 이 'annotation'을 신뢰하고 `String`으로 타입을 지정한다.  
그렇다면 'annotation' 중 어느 것으로도 주석 처리되지 않은 경우는 어떻게 될까?

```java
public class JavaTest {
    
  public String giveName() { /* ... */ }
}
```

Java에서는 모든 타입이 'null'이 될 수 있기에 Kotlin에서는 'nullable' 타입으로 안전하게 처리하는 것이 좋다.
그러나 실제 개발 환경에서는 개발자가 특정 변수나 객체가 절대 'null'이 아님을 알고 있는 상황이 많을 것이다.
이런 경우엔 Kotlin에서는 'not-null assertion `!!`'을 사용하여 해당 타입을 'non-nullable'로 명시적으로 나타낼 수 있다.

Java에서 제네릭 타입과 'annotation'이 없는 타입을 다룰 때 Kotlin에서 처리가 복잡해 질 수 있다. 
예를 들어, Java API가 'annotation' 없이 `List<User>`를 반환하는 경우, Kotlin에서는 기본적으로 'nullable'로 가정하기에 
해당 `List`와 `User`가 'null'이 아니라는 것을 알고 있어도 해당 리스트에 대해 'assertion'을 수행하고 'null'을 필터링 하는 작업을 해야 한다.

```java
public class UserRepo {
    public List<User> getUsers() { /* ... */ }
}
```

```kotlin
val users: List<User> = UserRepo().users!!.filterNotNull()
```

함수가 `List<List<User>>`인 경우 이는 더 복잡해질 것 이다.

```kotlin
val users: List<List<User>> = UserRepo().groupedUsers!!.map { it!!.filterNotNull() }
```

`List`의 경우 `map`이나 `filterNotNull`과 같은 함수를 통해 'null'을 체크 할 수 있다. 
하지만 다른 제네릭 타입의 'nullability'를 확인하는 것은 더 복잡해진다.
이러한 이유로 Kotlin에서 Java 코드와 상호 작용하면서 발생하는 'nullability' 문제를 해결하기 위해, 'platform type'이라는 개념을 도입했다. 
'platform type'은 Kotlin이 Java에서 가져온 타입의 'nullability' 여부가 명확하지 않을 때 사용하는 특별한 타입이다.

'platform type'은 `String!`과 같이 타입 이름 뒤에 느낌표(`!`)를 붙여 표기한다.  
그러나 이 표기법은 코드 내에서 직접 표기 할 수 없는데, 이는 'platform type'이 명시적으로 표기될 수 없는(non-denotable) 타입이기 때문이다. 
또한 'platform value'가 Kotlin 변수나 프로퍼티에 할당될 때 추론될 수 있지만, 명시적으로 설정될 수 없다.
대신 'nullable' 또는 'non-null' 중 예상하는 타입으로 선택할 수 있다.


```java
public class UserRepo {
    public User getUser() { /* ... */ }
}
```

```kotlin
val repo = UserRepo()
val user1 = repo.user           // Type of user1 is User!
val user2: User = repo.user     // Type of user2 is User
val user3: User? = repo.user    // Type of user3 is User? 
```

이처럼 Kotlin의 'platform type' 덕분에 Java로부터 제네릭 타입을 가져오는 것이 문제가 되지 않는다.

```kotlin
val users: List<User> = UserRepo().users
val users: List<List<User>> = UserRepo().groupedUsers
```

주의할 점으로 'non-null'로 가정한 것이 사실은 'null'일 수 있다는 위험성이 존재하기에, Java에서 Kotlin으로 'platform type'을 얻을 때 이런 위험성을 주의해야 한다.  

즉, 현재 함수가 'null'을 반환하지 않다고 해도 미래에도 변경되지 않을 것이라는 보장이 없다.
함수의 설계자가 'annotation'으로 표시하거나 주석으로 설명하지 않았다면 어떤 'contract'도 정해진 것이 없기에, 해당 함수의 행위는 미래에 변경 될 수 있다.

그렇기에 Kotlin과 상호 작용하는 Java 코드를 일부 제어할 수 있다면, 가능한 `@Nullable`과 `@NotNull` 'annotation'을 적극적으로 사용하는 것이 좋다.

```java
import org.jetbrains.annotations.NotNull;

public class UserRepo {
    public @NotNull User getUser() { /* ... */ }
}
```

Kotlin이 Android 개발에 'First-class citizen'으로 채택되면서, 
Android API는 Kotlin 개발자에게 더욱 우호적인 환경을 제공하기 위해 여러가지 중요한 변경 사항을 도입했고,
이 중 하나가 많이 노출된 타입에 `@Nullable`과 `@NotNull` 'annotation'을 추가하는 것이다.

Kotlin 코드에서 'platform type'을 다룰 때, 명시적으로 'nullable type'이나 'non-null type'으로 변환하는 것은 중요하다.
이는 Kotlin 코드의 안전성을 높이고 예기치 못한 'NPE'를 방지하는데 도움이 된다.

이를 확인하기 위해 아래 예제 `statedType`과 `platformType`의 동작 차이를 보자.

```java
public class JavaClass {
    public String getValue() { return null; }
}
```

```kotlin
fun statedType() {
    val value: String = JavaClass().value
    print(value.length)
}

fun platformType() {
    val value = JavaClass().value
    print(value.length)
}
```

두 상황 모두, 개발자는 `getValue()`가 'null'을 반환하지 않을 것이라 가정했지만, 
이는 잘못된 가정으로 두 경우 모두에서 'NPE'가 발생하고, 두 상황 모두 오류가 발생하는 위치에 차이가 있다.

`statedType`의 경우, 'NPE'는 Java로부터 값을 가져오는 것과 동일한 줄에서 발생한다.
우리가 'non-null type'을 잘못 가정했으며 'null'을 받았다는 것이 명확해진다. 
그렇다면 개발자는 단지 이 코드를 변경하고 코드의 나머지 부분을 이 변경에 맞게 **조정** 하면 된다.

`platformType`의 경우, 해당 값을 'non-null'로 사용하는 곳에서 'NPE'가 발생한다.  
'platform type'으로 타입이 지정된 변수는 'nullable'이 될 수도 있고 'non-null'이 될 수도 있다.
이런 변수는 몇 번은 안전하게 사용될 수 있겠지만, 이 후에 안전하지 않게 사용되어 'NPE'를 발생시킬 수 있다.
이런 변수를 사용할 때, '타입 시스템'은 개발자를 보호하지 못한다. 
이는 Java에서와 유사한 상황이지만, Kotlin에서는 객체를 단순하게 사용하는 것만으로 'NPE'가 발생한다고 예상하지 못한다.
누군가가 이를 안전하지 않게 사용할 가능성이 높으며, 그 결과 런타임 에러가 발생하고 그 원인을 찾기 어려울 수 있다.

```java
public class JavaClass {
    public String getValue() { return null; }
}
```

```kotlin
fun platformType() {
    val value = JavaClass().value
    print(value.length) // NPE
}

fun statedType() {
    val value: String = JavaClass().value // NPE
    print(value.length) 
}
```

더 위험한 것은, 'platform type'을 시스템 전체적으로 전파되는 상황이다.  
예를 들어, 'platform type'을 인터페이스의 일부로 사용하는 경우 안정성과 명확성 문제가 전체 시스템에 영향을 미칠 수 있다.

```kotlin
interface UserRepo {
    fun getUserName() = JavaClass().value
}
```

'platform type'의 추론된 타입이 의미하는 바는, 개발자가 해당 타입을 'nullable type'으로 처리할 것인지, 'non-null type'으로 처리할 것인지를 선택할 수 있다는 것이다.
어떤 개발자는 초기화 시점에서 이를 'nullable'로 처리할 수 있고, 또 어떤 개발자는 사용 위치에서 'non-null'로 처리할 수 있다.

```kotlin
class RepoImpl: UserRepo {
    override fun getUserName(): String? { return null }
}

fun main() {
    val repo: UserRepo = RepoImpl()
    val text: String = repo.getUserName() // NPE in runtime
    
    print("User name length is ${text.length}")
}
```

이와 같이 'platform type'을 코드 내에 전파하는 것은 여러 문제를 일으킬 수 있으며, 특히 'NPE'의 위험을 증가시킨다.
'platform type'은 nullability가 명확하지 않기에, 이를 사용하는 코드는 예상치 못한 오류를 발생시킬 수 있다.
따라서, 안전상의 이유로 'platform type'을 가능한 한 빨리 제거하는 것이 좋다.
이 경우, IDE에서 경고를 통해 개발자들을 도와준다.