# Kotlin의 플랫폼 타입과 Null-safety

Kotlin은 Null-safety를 훌륭하게 처리하므로 `Null-Pointer Exceptions`를 거의 없애준다.  
특히 Java 및 C와 같은 언어와 비교하면 더 돋보인다.  
그러나 Kotlin과 `null-safety`가 확실하지 않은 언어와의 연결은 완전히 보장되지 않는다.  

Java 메서드가 반환 유형으로 `String`을 선언한다고 가정하면 Kotlin에서는 어떤 유형으로 변환 될까?

---

## Java에서 Kotlin으로 타입 연결

- `@Nullable` 주석이 있는 경우, 이것을 `nullable`로 가정하고 `String?`로 해석
- `@NotNull` 주석이 있는 경우, `String`으로 타입을 설정
- 주석이 없는 경우 Kotlin에서는 Java로부터 오는 유형을 기본적으로 `nullable`로 가정하지 않습니다.  
  대신, `플랫폼 타입`이라는 특수한 타입으로 처리합니다.

### 플랫폼 타입
- 다른 언어에서 오는 nullability가 알려지지 않은 타입을 말한다.  
- Kotlin 변수나 속성에 플랫폼 값이 할당되면 추론될 수 있지만 명시적으로 설정할 수는 없다.
- `String!`처럼 `!`를 붙여 표기한다.

```java
public class UserRepository {
   public User getUser() { ... } 
}
```

```kotlin
val repository = UserRepository()
val user1 = repository.user // user1 : User!
val user2: User = repository.user // user2 : User
val user3: User? = repository.user // user3: User?
```

#### 플랫폼 타입의 위험성   
- 우리가 `null`이 아님을 가정한 것이 `null`일 수 있기 때문에 Java로부터 플랫폼 타입을 얻을 때는 항상 조심해야 한다.

#### 플랫폼 타입 제거
- 보안상의 이유로 플랫폼 타입을 가능한 한 빨리 제거하는 것이 좋다. 플랫폼 타입을 제거하지 않고 그대로 사용할 경우  
  그 값이 `null`인지 아닌지에 따라 예상치 못한 `Null-Pointer Exception`을 발생시킬 수 있기 때문이다.

### Java -> Kotlin 타입 변경 예시

`JavaClass`에서 `getValue()` 메소드는 `String`을 반환하지만 실제로는 `null` 값을 반환한다.   
이를 Kotlin에서 처리할 때 2가지 방법을 사용한다.

#### 1) 명시적 타입 선언 (statedType): 값을 Java로부터 가져오는 것과 동일한 줄에서 `NPE`가 발생

이는 우리가 `not-null` 타입을 잘못 가정했으며 `null`을 얻었다는 것이 분명하게 나타나며,  
이를 변경하고 코드의 나머지 부분을 이 변경에 맞게 **조정** 해야한다.

```kotlin
fun statedType() {
    val value: String = JavaClass().value // NPE 발생
    println(value.length)
}
```

#### 2) 플랫폼 타입 (platformType): 값이 `not-nullable`로 사용될 때 `NPE`가 발생합니다. 이 경우 `NPE`의 원인을 찾기가 어렵습니다.

```kotlin
fun platformType() {
    val value = JavaClass().value // 값에 NPE는 없지만
    println(value.length) // NPE 발생
}
```

따라서, 플랫폼 타입은 명시적으로 `null` 처리를 해주는 것이 좋다.   
이는 보안상의 이유뿐만 아니라, 예외 처리를 더 명확하게 하기 위해서도 중요하다.