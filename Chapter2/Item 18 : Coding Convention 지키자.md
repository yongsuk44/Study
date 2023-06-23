# 항목 18: Coding Convention 준수

---

Kotlin은 잘 정리된 규칙을 [Coding Conventions 섹션](https://kotlinlang.org/docs/coding-conventions.html#verify-that-your-code-follows-the-style-guide)에 명시하고 있습니다.   
이 규칙들이 모든 프로젝트에 최적화된 것은 아니지만, 프로젝트에서 존중되는 공통된 규칙을 가지는 것이 이상적입니다. 

위 Coding Convention을 지키면 다음과 같은 장점들이 있습니다.

- 프로젝트 간 이동이 쉬워집니다.
- 외부 개발자도 코드를 더 쉽게 읽을 수 있습니다.
- 코드의 작동 방식을 추론하기가 쉬워집니다.
- 공통 저장소와의 코드 병합이나 한 프로젝트에서 다른 프로젝트로 코드 일부를 이동하는 것이 용이해집니다.

## Coding Convention 익숙해지기

개발자들은 Coding Convention에 익숙해져야 하며, 규칙이 시간이 지나며 변경될 경우에도 적응해야 합니다.   
이 두 가지를 동시에 적응하는것이 어렵겠지만, 다음과 같은 도구가 도움이 될 수 있습니다.

- IntelliJ 포맷터: 공식 Coding Conventions 스타일에 따라 자동으로 코드를 포맷팅하도록 설정할 수 있습니다. 
  이렇게 설정하려면 Settings → Editor → Code Style → Kotlin로 이동하여, 
  오른쪽 상단의 `Set from…` 링크를 클릭하고 메뉴에서 `Predefined style / Kotlin style guide`를 선택하면 됩니다.
- ktlint: 코드를 분석하여 Coding Convention에 위반되는 코드를 알려주는 linter입니다.

## Kotlin 프로젝트의 일관성

Kotlin 프로젝트를 살펴보면, 대부분이 직관적으로 대부분의 규칙과 일관성이 있는 것을 확인할 수 있습니다. 
이는 Kotlin이 주로 Java 코딩 규칙을 따르고, 현재의 Kotlin 개발자 대부분이 Java 개발자 출신이기 때문일 것입니다.  
하지만 종종 어겨지는 규칙 중 하나는 클래스와 함수의 포맷팅 방식입니다. 

Convention에 따르면, 다음과 같이 짧은 생성자를 가진 클래스는 한 줄로 정의될 수 있습니다.

```kotlin
class FullName(val name: String, val surName: String)
```

그러나 많은 매개변수를 가진 클래스는 각 매개변수가 다른 줄에 오도록 포맷팅되어야 하며, 첫 번째 줄에는 매개변수가 없어야 합니다.

```kotlin
class Person(
    val id: Int = 0,
    val name: String = "",
    val surName: String = ""
) : Human(id, name) {
    // body 
}
```

이는 긴 함수를 포맷팅하는 방식과 동일합니다.

```kotlin
public fun <T> Iterable<T>.joinToString(
    separator: CharSequence = ", ",
    prefix: CharSequence = "",
    postfix: CharSequence = "",
    limit: Int = -1,
    truncated: CharSequence = "...",
    transform: ((T) -> CharSequence)? = null
): String {
    // body
}
```

## 잘못된 포맷팅의 위험성

Coding Convention에 따르면 각 매개변수는 새로운 줄에 오도록 하지만, 
첫 번째 매개변수를 같은 줄에 두고 그 다음 모든 매개변수를 들여 쓰는 방식은 허용되지 않습니다.

```kotlin
// 이런 식으로 작성 하지마세요.
class Person(val id: Int = 0,
             val name: String = "",
             val surName: String = "") : Human(id, name) {
    // body 
}
```
위와 같은 코드 스타일은 2가지 문제를 가질 수 있습니다.

### Indentation(들여 쓰기)의 불일관성
각 클래스의 시작에서 Indentation이 달라질 수 있다는 것입니다.
첫 번째 매개변수를 같은 줄에 배치하면, 해당 클래스의 이름이나 크기에 따라 매개변수의 들여쓰기 수준이 달라집니다.
이는 코드의 일관성을 해칠 수 있습니다. 

게다가 클래스 이름이 변경될 때마다 모든 주 생성자 매개변수의 들여쓰기를 조정해야 합니다.
이는 유지 보수를 어렵게 만들고 실수의 여지가 늘어납니다.

### 코드 가독성 저하
위 방식으로 작성된 클래스가 너무 넓어지는 경향을 보일 수 있습니다.
클래스의 너비는 클래스의 이름과 `class` 키워드, 가장 긴 주 생성자 매개변수 또는 마지막 매개변수, 슈퍼클래스 및 인터페이스를 합한 것이 됩니다.

이로 인해 코드가 가로로 길어지며, 이는 가독성으 저하 시킵니다.
특히, 코드를 작성하거나 검토할 때 화면을 가로로 스크롤해야 하는 경우 이 문제가 더 부각이 됩니다.


## 프로젝트별 코딩 규칙 준수

일부 팀은 약간 다른 Coding Convention을 사용하기로 결정할 수 있습니다. 
이는 하나의 프로젝트로 보면 괜찮겠지만, 이러한 Coding Convention은 회사 전체 프로젝트를 담당하는 모든 개발자들이 이해할 수 있어야 합니다.