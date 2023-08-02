# Item 47: inline class 사용 고려 

함수만 `inline`화 할 수 있는것이 아니며, 단일 값을 가진 객체도 이 값으로 대체될 수 있습니다.
이러한 기능을 가능하게 하려면 단일 생성자 속성이 있는 클래스 앞에 `inline` 수식어를 붙여야 합니다.

```kotlin
inline class Name(private val value: String) {
    // ...
}

val name: Name = Name("John")

// 컴파일 중 아래와 같이 유사하게 변경
val name: String = "John"
```

`inline` 클래스의 메서드들은 정적메서드로 정의됩니다.

```kotlin
inline class Name(private val value: String) {
    // ...
    fun greet() = print("Hello, I am $value")
}

val name: Name = Name("John")
name.greet()

// 컴파일 중 아래와 같이 유사하게 변경
val name: String = "John"
Name.`greet-impl`(name)
```

`inline class`를 통해 일부 유형(위 `String`과 같은)을 래핑할 수 있으며 성능 오버헤드 없이 사용할 수 있습니다.

`inline class`는 2가지 용도로 사용됩니다.
1. 측정 단위 나타내기 
2. 타입을 사용하여 사용자 오용 방지